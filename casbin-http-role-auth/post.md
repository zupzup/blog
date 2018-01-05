Authentication and Authorization are essential parts of any secured Web Application. I recently finished writing my first serious web application in Go and this is one part in a series of posts of things about what I learned during that experience.

In this post, we will look at HTTP `Authorization` in Go using the great [casbin](https://github.com/casbin/casbin) library. We will also use [scs](https://github.com/alexedwards/scs) for session management in the code example.

The following example is very basic, but I hope it shows how implementing authorization can be achieved in a Go web application. The focus is on the usage of `casbin`, so the other parts of the application will be very minimal and abstract (e.g.: a login without password).

So, let's take a look! 

*Disclaimer: Please don't use the following example code as a template for a production-grade application, the code focuses on clarity, not on security.*

## Setup 

First, we create a `User` model, with some utility functions:

```go
type User struct {
    ID   int
    Name string
    Role string
}

type Users []User

func (u Users) Exists(id int) bool {
    ...
}

func (u Users) FindByName(name string) (User, error) {
    ...
}
```

Then we come to the `casbin` configuration. There are two files we will create for this purpose - a configuration file and a policy file. The configuration file uses the `PERM` metamodel. `PERM` stands for Policy, Effect, Request, Matchers. 

Our `auth_model.conf` has the following contents:

```
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = r.sub == p.sub && keyMatch(r.obj, p.obj) && (r.act == p.act || p.act == "*")
```

This defines the request and policy to contain `subject`, `object` and `action`. In the HTTP case, this means that the subject is the user role, the object is the path the user wants to access and the action is the request method (e.g.: GET, POST etc.).

The `matchers` define how the policy parts are matched, either directly like the subject or with a helper method like `keyMatch`, which can also match wildcards. `casbin` is of course a lot more powerful than this simple example lets on. You can define all sorts of custom functionality in a declarative way, making it easy to switch and maintain authorization configurations.

When security is concerned, I usually opt for the simplest solution possible, because when things get complex and unmaintainable, errors start to happen.

The policy file, in this example, is a simple `csv` file describing which role can access which paths etc.

The `policy.csv` file looks like this:

```
p, admin, /*, *
p, anonymous, /login, *
p, member, /logout, *
p, member, /member/*, *
```

This is, of course, also very simple. In this case, we simply define that the `admin` role can access everything, the `member` role can access everything after `/member/` as well as `logout` and all unauthenticated users can use the `login`.

What I like about this format is that it stays maintainable, even if there are many rules and user roles.

## Implementation

Let's start with the `main` function, which sets everything up and starts the HTTP server:

```go
func main() {
    // setup casbin auth rules
    authEnforcer, err := casbin.NewEnforcerSafe("./auth_model.conf", "./policy.csv")
    if err != nil {
        log.Fatal(err)
    }

    // setup session store
    engine := memstore.New(30 * time.Minute)
    sessionManager := session.Manage(engine, session.IdleTimeout(30*time.Minute), session.Persist(true), session.Secure(true))

    // setup users
    users := createUsers()

    // setup routes
    mux := http.NewServeMux()
    mux.HandleFunc("/login", loginHandler(users))
    mux.HandleFunc("/logout", logoutHandler())
    mux.HandleFunc("/member/current", currentMemberHandler())
    mux.HandleFunc("/member/role", memberRoleHandler())
    mux.HandleFunc("/admin/stuff", adminHandler())

    log.Print("Server started on localhost:8080")
    log.Fatal(http.ListenAndServe(":8080", sessionManager(authorization.Authorizer(authEnforcer, users)(mux))))

}
```

There are several things going on here. Basically, we set up the authorization rules, session management, users, HTTP handlers and start the HTTP server with the router wrapped by an authorization middleware and the session manager.

Let's go through it one by one.

First, we create a `casbin` enforcer with the above mentioned `auth_model.conf` and `policy.csv` files. If this fails, we shut down, as there is likely something wrong with the authorization rules.

The next step is to set up the `sessionManager`. We create an in-memory session store with a 30-minute timeout and a session manager with secure cookie storage. 

The `createUsers` function simply creates three users with their user roles as seen below:

```go
func createUsers() model.Users {
    users := model.Users{}
    users = append(users, model.User{ID: 1, Name: "Admin", Role: "admin"})
    users = append(users, model.User{ID: 2, Name: "Sabine", Role: "member"})
    users = append(users, model.User{ID: 3, Name: "Sepp", Role: "member"})
    return users
}
```

In a real-world application, we would probably have a database containing these users - in this example we will just use this list.

Next up are the handlers for `login` and `logout`:

```go
func loginHandler(users model.Users) http.HandlerFunc {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        name := r.PostFormValue("name")
        user, err := users.FindByName(name)
        if err != nil {
            writeError(http.StatusBadRequest, "WRONG_CREDENTIALS", w, err)
            return
        }
        // setup session
        if err := session.RegenerateToken(r); err != nil {
            writeError(http.StatusInternalServerError, "ERROR", w, err)
            return
        }
        session.PutInt(r, "userID", user.ID)
        session.PutString(r, "role", user.Role)
        writeSuccess("SUCCESS", w)
    })
}

func logoutHandler() http.HandlerFunc {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if err := session.Renew(r); err != nil {
            writeError(http.StatusInternalServerError, "ERROR", w, err)
            return
        }
        writeSuccess("SUCCESS", w)
    })
}
```

For `login`, we get the `name` from the request payload, check if there is a user with that name and if that is the case, we create a new session and put the user role and ID into it.

The `logout` handler simply creates a new, empty session for the user, deleting the old session from the session store, logging out the user.

Then we have a few handlers, which test the implementation by returning the userID and user role. The endpoints for these handlers are secured by `casbin` as seen above in the `policy.csv` file.

```go
func currentMemberHandler() http.HandlerFunc {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        uid, err := session.GetInt(r, "userID")
        if err != nil {
            writeError(http.StatusInternalServerError, "ERROR", w, err)
            return
        }
        writeSuccess(fmt.Sprintf("User with ID: %d", uid), w)
    })
}

func memberRoleHandler() http.HandlerFunc {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        role, err := session.GetString(r, "role")
        if err != nil {
            writeError(http.StatusInternalServerError, "ERROR", w, err)
            return
        }
        writeSuccess(fmt.Sprintf("User with Role: %s", role), w)
    })
}

func adminHandler() http.HandlerFunc {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        writeSuccess("I'm an Admin!", w)
    })
}
```

With `session.GetInt` and `session.GetString` we can fetch values stored in the current session.

For these handlers to be actually secured by the authorization mechanism, we need to implement the `Authorizer` middleware, which wraps the router.

```go
func Authorizer(e *casbin.Enforcer, users model.Users) func(next http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        fn := func(w http.ResponseWriter, r *http.Request) {
            role, err := session.GetString(r, "role")
            if err != nil {
                writeError(http.StatusInternalServerError, "ERROR", w, err)
                return
            }

            if role == "" {
                role = "anonymous"
            }

            // if it's a member, check if the user still exists
            if role == "member" {
                uid, err := session.GetInt(r, "userID")
                if err != nil {
                    writeError(http.StatusInternalServerError, "ERROR", w, err)
                    return
                }
                exists := users.Exists(uid)
                if !exists {
                    writeError(http.StatusForbidden, "FORBIDDEN", w, errors.New("user does not exist"))
                    return
                }
            }

            // casbin rule enforcing
            res, err := e.EnforceSafe(role, r.URL.Path, r.Method)
            if err != nil {
                writeError(http.StatusInternalServerError, "ERROR", w, err)
                return
            }
            if res {
                next.ServeHTTP(w, r)
            } else {
                writeError(http.StatusForbidden, "FORBIDDEN", w, errors.New("unauthorized"))
                return
            }
        }

        return http.HandlerFunc(fn)
    }
}
```

The `Authorizer` middleware receives the `casbin` rule enforcer and the users as parameters. First, it tries to fetch the role of the requesting user from the session.

If the user has no user role, we set it to `anonymous`, otherwise, if the user is a `member` we check if it's a valid account that still exists by fetching the `userID` from the session and checking it against the `users` list.

After these preliminary checks, we can give the user role, request path and the request method to the `casbin` enforcer, which decides, if the user with the given role (`subject`) is allowed to access the action specified by the request method (`action`) and the path (`object`).

If the check fails, we return a `403` error, otherwise we simply call the wrapped HTTP handler, authorizing the user to execute this action.

As mentioned above in the `main` method, the `sessionManager` and `Authorizer` wrap our `mux`, so that every request needs to go through this middleware, making sure that all requests are secured with our authorization mechanism.

You can test this by logging in with the different users and trying out the above-mentioned handlers using a tool like `curl` or `postman`.

That's it. You can find the full code [here](https://github.com/zupzup/casbin-http-role-example).

## Conclusion 

I have used `casbin` in production in a medium-sized web application and was pleased with the maintainability and stability of the library. Looking over its documentation, `casbin` seems like a very powerful tool for authorization, featuring a staggering amount of access control models in a declarative way.

I hope this example could showcase the power of `casbin` and `scs` and, again, to demonstrate the conciseness and clarity of Go for web applications and in general.

Have fun authorizing. ;)

#### Resources

* [Full Sample Code on Github](https://github.com/zupzup/casbin-http-role-example)
* [casbin](https://github.com/casbin/casbin)
* [scs](https://github.com/alexedwards/scs)
