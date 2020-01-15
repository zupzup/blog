When implementing web services, the need to run recurring jobs often arises at some point (e.g. for cleanup / maintenance, or state-sharing purposes). Depending on the setup, these jobs might either run on dedicated services, or on the web services themselves.

In this post, we'll look at a basic implementation of a Job Runner, which synchronizes job runs between instances of the services (so we won't run jobs more than once at the same time) and which executes the jobs asynchronously using tokio.

It will be possible to register new jobs with a schedule for running them at the job runner. The job runner itself will simply check for jobs to be ran at a configurable interval and, when a job is being run, put an entry for this job into a shared `redis` cache.

This redis cache is used to synchronize, so other instances of the job runner don't run the same job at the same time, or too often in general.

We'll go over the implementation step by step and at the end there'll be a quick example of how to integrate this with an existing web service.

## Implementation 

First, let's define the API to create the job-runner and to register jobs with it. We'll need some sort of context object to provide things to our jobs such as DB connections:

```rust
let job_context = JobContext {
	db_pool: db_pool.clone(),
	...
};
```

Also, we'd like to define jobs without much fuss, so let's see how a `Job` trait could look like:

```rust
pub trait Job {
    fn run(&self, ctx: &JobContext) -> JobResult;
    fn get_interval(&self) -> Duration;
    fn get_name(&self) -> &'static str;
    fn get_sync_key(&self) -> &'static str;
    fn box_clone(&self) -> BoxedJob;
}
```

All jobs need to implement this trait. It includes the job's name, the interval it's supposed to run on. It also returns the `sync_key`, which is the key within redis used to synchronize
the run of this job between instances.

We also need to implement a `box_clone` method, which is basically just this:

```rust
fn box_clone(&self) -> BoxedJob {
	Box::new((*self).clone())
}
```

I haven't found a nicer way 


There are two custom types in the above snippet - BoxedJob and JobResult:

```rust
type Result<T> = std::result::Result<T, Error>; // Error is our custom error type
pub type BoxedJob = Box<dyn Job + Send + Sync>;
type JobResult = BoxFuture<'static, Result<()>>;
```

BoxedJob is simply a boxed up job we can pass between threads safely. A JobResult is what we expect a job run to return and in this case, it's simply a future returning a `Result<()>`,
as we're not interested in any return values, but only whether the job succeeded or not.

Alright, so let's see how such a Job implementation could look like:

```rust
#[derive(Clone)]
pub struct DummyJob;

impl Job for DummyJob {
    fn run(&self, ctx: &JobContext) -> JobResult {
        info!("Running Dummyjob...");
        let ctx_clone = ctx.clone();
        let fut = async move {
            let mut db = ctx_clone.db_pool.get().await.map_err(DBPoolError)?;
			// do something with the DB connection...
            Ok(())
        };
        Box::pin(fut)
    }
    fn get_interval(&self) -> Duration {
        Duration::from_secs(6000)
    }
    fn get_name(&self) -> &'static str {
        DUMMY_JOB_NAME
    }
    fn get_sync_key(&self) -> &'static str {
        DUMMY_JOB_SYNC_CACHE_KEY
    }
    fn box_clone(&self) -> BoxedJob {
        Box::new((*self).clone())
    }
}
```

Then, setting up our runner is rather simple:


```rust
let dummy_job = DummyJob {};
let job_runner = JobRunner::new(
	redis_pool.clone(),
	job_context,
	vec![
		Box::new(dummy_job) as BoxedJob,
	],
);

job_runner.run_jobs().await
```

We instantiate a `DummyJob` instance and a `JobRunner`, pass our redis pool to it, the above defined job context and a list of jobs, which includes our boxed up dummy job.

The `redis_pool` and `db_pool` mentioned above are simple redis and postgres pools using [mobc](https://github.com/importcjj/mobc), but you can use any pool you like, as long as it can be shared across threads.

Ok, so this is the API we're aiming for. Let's start building the `JobRunner`.

```rust
pub struct JobRunner {
    jobs: Vec<BoxedJob>,
    redis_pool: RedisPool,
    job_context: JobContext,
}
```

So far so good. The `JobRunner` holds the above mentioned redis pool and job context, as well as a list of Jobs. These jobs are a boxed trait object so we can use different ones in the same list.

The first function we'll look at is `run_jobs`. This is the initial function call, which kicks off the whole shebang.
```rust
pub async fn run_jobs(self) -> Result<()> {
	self.announce_jobs();
	let mut job_interval =
		tokio::time::interval(Duration::from_secs(JOB_CHECKING_INTERVAL_SECS));
	let arc_jobs = Arc::new(&self.jobs);
	loop {
		job_interval.tick().await;
		match self.check_and_run_jobs(&arc_jobs).await {
			Ok(_) => (),
			Err(e) => error!("Could not check and run Jobs: {}", e),
		};
	}
}
```

In this case, we'll first announce the registered jobs, logging them and their running interval. Then, we check if a job needs to be executed in an interval.

This interval also controls how exact our intervals are going to be. For example, if we check for pending job runs ever 2 minutes, it won't make sense to schedule a job every 30 seconds. In general, if it's not particularly critical that jobs are run at a very specific time, turning this interval time up is a good idea, as it will execute less often and hence have less of an impact on your service's runtime performance.

Anyway, ever $interval seconds, we call `check_and_run_jobs`, which we'll look at next.


```rust
async fn check_and_run_jobs(&self, arc_jobs: &Arc<&Vec<BoxedJob>>) -> Result<()> {
	let jobs = arc_jobs.clone();
	for job in jobs.iter() {
		match self.check_and_run_job(job, &self.redis_pool).await {
			Ok(_) => (),
			Err(e) => error!("Error during Job run: {}", e),
		};
	}
	Ok(())
}
```

Alright, in this function, we iterate all registered jobs and call `check_and_run_job` for each one of them. What's interesting here, is that we moved our list of jobs inside of an `Arc`, in order to be able to safely share it between threads.

For each job, we do the following:

```rust
async fn check_and_run_job(&self, job: &BoxedJob, redis_pool: &RedisPool) -> Result<()> {
	let now = Utc::now().timestamp() as u64;
	let j = job.box_clone();
	let job_context_clone = self.job_context.clone();
	match cache::get_str(&redis_pool, job.get_sync_key()).await {
		Ok(v) => {
			let last_run = v.parse::<u64>().map_err(|_| ParseLastJobRunError(v))?;
			if now > job.get_interval().as_secs() + last_run {
				self.set_last_run(now, &redis_pool, job.get_sync_key())
					.await;
				self.run_job(j, job_context_clone);
			}
		}
		Err(_) => {
			self.set_last_run(now, &redis_pool, job.get_sync_key())
				.await;
			self.run_job(j, job_context_clone);
		}
	};
	Ok(())
}
```

Ok, now it's getting a lot more interesting. First, we check the shared redis cache, if we already have an entry for this job, if not, we run the job and set the `last_run_date` to now. If we do have a value, we check if it's time to run it (`now` needs to be greater than `last_run` + `job_interval`). If it is, we run it, otherwise we bail and try again later.

The `set_last_run` method simply writes the last run time to the shared redis-cache.

Running the job itself isn't particularly exciting:

```rust
fn run_job(&self, job: BoxedJob, job_context: JobContext) {
	let job_name = job.get_name();
	tokio::spawn(async move {
		info!("Starting Job {}...", job_name);
		match job.run(&job_context).await {
			Ok(_) => info!("Job {} finished successfully", job_name),
			Err(e) => error!("Job {} finished with error: {}", job_name, e),
		};
	});
}
```

Because we are using the `BoxedJob` trait object, we can just call `job.run` with the given context. We do this within `tokio::spawn`, so the actual job is run on the tokio threadpool. After the job was executed, we also report success or failure using a log statement.

Alright. That's all the infrastructure we need, now we can try and integrate this with an HTTP server. In our case we'll just use `hyper`.

```rust
// first, define job context
let job_context = JobContext {
	db_pool: db_pool.clone(),
};

// then, instantiate the job implemented above
let dummy_job = DummyJob {};
let job_runner = JobRunner::new(
	redis_pool.clone(),
	job_context,
	vec![
		Box::new(dummy_job) as BoxedJob,
	],
);

// prepare hyper server
let new_service = make_service_fn(move |_| {
	... service routing logic ...
});
let addr = format!("0.0.0.0:{}", 8080).parse().unwrap();
let server = Server::bind(&addr).serve(new_service);
info!("Listening on http://{}", addr);

// start the hyper server and our job runner at the same time
let res = join!(server, job_runner.run_jobs());

res.0.map_err(ServerCrashedError)?;
Ok(())
```

So, we first define our job context with the `db_pool` we need in our DummyJob defined above. Then, we instantiate the Job and the JobRunner, moving DummyJob and the JobContext inside the JobRunner.

Afterwards, we prepare a basic hyper server and run both concurrently using `join!`. If any of them crash, we handle the error.

If we run this now, we can access the running server and we can see the job running in the log.

// TODO: build minimal running example on github (with docker commands for redis etc.) and adapt above code based on it


```rust
```

```rust
```


```rust
```

The full example code can be found [here](TODO)

## Conclusion 

TBD

#### Resources

* [Code Example](TODO)
* [mobc](https://github.com/importcjj/mobc
* [hyper](https://hyper.rs/)
