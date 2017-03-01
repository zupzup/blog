About two months ago I wrote [this](https://zupzup.org/static-deploy-tool/) post describing how and why I built [static-aws-deploy](https://github.com/zupzup/static-aws-deploy), which I've been using to update my blog since then.

So far the tool has worked well, but I recently noticed that the number of files to upload each time grows steadily and I decided to tackle **delta-uploading** before it gets out of hand.

A delta-upload is just a fancy way of saying *only upload files which have changed or are new* - so instead of updating all the files on my blog every time I publish a post, I only want to upload the new files and the files which have changes (e.g.: the RSS feed), which will save a bit of bandwidth, lots of requests to the API and will make the tool more flexible for small changes like fixing typos after publishing.

Let's take a look at what I came up with!

## Approach 

There are several ways of finding out which files are new or changed. In this case, I decided to use the `lastModifiedDate` of the files, as well as their `md5` hash. A file will be marked as new or changed, if the `lastModifiedDate` of the local file is after the file's which is already uploaded AND if the `md5` hashes don't match up.

The reason why I used `md5` is simply because the [S3 REST API](http://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html) has a way of fetching all files within a bucket, which is documented [here](http://docs.aws.amazon.com/AmazonS3/latest/API/v2-RESTBucketGET.html), which also includes an `ETag` property, which is the `md5` hash of the file.

From an architectural perspective, [static-aws-deploy](https://github.com/zupzup/static-aws-deploy) already walks all the files in order to collect them and generate their `Headers` for uploading, so hooking in there seemed like a good idea.

## Implementation 

First, I needed to add a flag to enable **delta-uploading** to keep backwards compatibility:

```go
var (
    ...
    delta      bool
)

flag.BoolVar(&delta, "delta", false, "only upload changed files")
flag.BoolVar(&delta, "d", false, "only upload changed files (shorthand)")
```

The next step was to pass this flag to the `upload.ParseFiles` function. 

```go
files, err := upload.ParseFiles(&config.S3, delta)
```

In this function, if the `delta` flag is set, a request needs to be made to AWS S3 to fetch infos about all files within the bucket and put them into a nice data structure for later use. The datastructure looks like this:

```go
type Delta map[string]*DeltaProperties

type DeltaProperties struct {
    LastModified time.Time
    ETag         string
}
```

Basically, it just maps from a file to its properties we need to determine whether it has been changed or not. In order to create and populate this map, I wrote the `getDeltaMap` function, which requests AWS S3, parses the XML with [etree](https://github.com/beevik/etree) and returns a `Delta` and handles all errors.

it looks something like this (error handling omitted for brevity):

```go
func getDeltaMap(config *Config) (Delta, error) {
    // make authenticated AWS S3 request
    client := &http.Client{}
    req, err := http.NewRequest("GET", fmt.Sprintf("https://s3.amazonaws.com/%s/?list-type=2", config.Bucket.Name), nil)
    awsauth.Sign(req, awsauth.Credentials{
        AccessKeyID:     config.Bucket.Accesskey,
        SecretAccessKey: config.Bucket.Key,
    })
    resp, err := client.Do(req)

    // parse xml
    doc := etree.NewDocument()
    doc.ReadFrom(resp.Body);
    root := doc.SelectElement("ListBucketResult")
    contents := root.SelectElements("Contents")

    // build map
    deltaMap := make(Delta)
    for _, file := range contents {
        lastModified := file.SelectElement("LastModified")
        etag := file.SelectElement("ETag")
        key := file.SelectElement("Key")
        parsedLastModified, err := time.Parse(time.RFC3339Nano, lastModified.Text())
        deltaProp := DeltaProperties{
            ETag:           strings.Trim(etag.Text(), "\""),
            LastModified:   parsedLastModified,
        }
        deltaMap[key.Text()] = &deltaProp
    }
    return deltaMap, nil
}
```

Nothing too surprising here, except that we need to `Trim` some `""`s from the `ETag`, because the XML is like that. With this in place, we already have the key component for our **delta-upload** - a way to check if the files we are planning to upload are different from the ones already on the server.

The next step was to find out the `lastModifiedDate` and `md5` hash of our local files. For this purpose, I created the `hasFileChanged` function as follows:

```go
func hasFileChanged(info os.FileInfo, deltaMap Delta, uploadPath, filePath string) (bool, error) {
    etag, err := calculateETag(filePath)
    if err != nil {
        return false, err
    }
    deltaProps := deltaMap[uploadPath]
    if deltaProps != nil {
        return etag != deltaProps.ETag && info.ModTime().After(deltaProps.LastModified), nil
    }
    return true, nil
}
```

This function will be called while walking all the files to be uploaded. It gets an `os.FileInfo`, the `deltaMap` and two paths. One of these paths is the normal, local filepath we need to calculate the `md5` hash and the other one is the *cleaned* path, which matches the `Key` of the files already uploaded which are mapped within `deltaMap`.

First, the `ETag` of the local file is calculated:

```go
func calculateETag(path string) (string, error) {
    bytes, err := ioutil.ReadFile(path)
    if err != nil {
        return "", fmt.Errorf("could not read file: %s while calculating it's ETag, %v", path, err)
    }
    return fmt.Sprintf("%x", md5.Sum(bytes)), nil
}
```

And then, for each file within `deltaMap`, we check if the `ETag`s match and if the file was modified after it was uploaded. This way we will mark both changed and new files.

Armed with these functions, it was trivial to extend the file-walking logic:

```go
// in ParseFiles(...)
...
deltaMap := make(Delta)
if delta {
    deltaMap, err = getDeltaMap(config)
    if err != nil {
        return nil, err
    }
}
err = filepath.Walk(source, func(path string, info os.FileInfo, err error) error {
    if !info.IsDir() && !re.MatchString(path) {
        hasChanged := true
        if delta {
            hasChanged, err = hasFileChanged(info, deltaMap, getUploadPath(config, path), path)
            if err != nil {
                return err
            }
        }
        if hasChanged {
            result[path] = []Header{}
        }
    }
    return nil
})
```

Alright, that's it. Now running `static-aws-deploy --dr -d` on my blog source reveals the following:

```
0 Files to upload (4 concurrently)...
Delta Dry Run finished.
```

And after changing the RSS-feed (index.xml) manually:

```
1 Files to upload (4 concurrently)...
www/index.xml...Done.
Delta Dry Run finished.
```

Done!

You can check out the full changelog of the commit [here](https://github.com/zupzup/static-aws-deploy/commit/a8a4b120d834f416254018c79cfd4fcba0b9c7c4).

## Conclusion

A fun little feature for a useful little tool - nice! 

This post has already been successfully uploaded using this mechanism, which will hopefully save time, bandwidth and requests for many posts to come! ;)


#### Resources

* [static-aws-deploy](https://github.com/zupzup/static-aws-deploy)
* [changelog of this implementation](https://github.com/zupzup/static-aws-deploy/commit/a8a4b120d834f416254018c79cfd4fcba0b9c7c4)
* [initial blog post](https://zupzup.org/static-deploy-tool/)
* [S3 REST API](http://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)
