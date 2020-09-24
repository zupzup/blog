I recently finished writing my first serious web application in Go and there were a few things which stood out in regards to web development with Go. In the past, I developed web applications as a profession and as a hobby in a multitude of different languages and paradigms, so I have seen quite a few things in this area.

Overall I liked using Go for web development, although it took some getting used to at first. Some things were a bit clunky but at other parts, such as with uploading and downloading files as described in this short article, Go's standard library and inherent simplicity made it a very enjoyable experience.

In the next few articles I will focus on some of the things I encountered writing a production-grade web application in Go, especially regarding authentication/authorization.

This post will show a basic example of HTTP File Upload and Download. Imagine an HTML form with a `type` text input and an `uploadFile` file input as a client.

Let's see how Go can solve this omnipresent problem of web engineering.

## Code Example

First, we set up the web server with our two routes `/upload` for the File Upload and `/files/*` for the File Download.

```go
const maxUploadSize = 2 * 1024 * 1024 // 2 MB 
const uploadPath = "./tmp"

func main() {
    http.HandleFunc("/upload", uploadFileHandler())

    fs := http.FileServer(http.Dir(uploadPath))
    http.Handle("/files/", http.StripPrefix("/files", fs))

    log.Print("Server started on localhost:8080, use /upload for uploading files and /files/{fileName} for downloading files.")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

We also define constants for the target directory to upload to as well as the maximum file-size we accept. Notice here, how simple the whole concept of serving files is - using only tools from the standard library we use `http.FileServer` to create an HTTP handler, which serves the files from the directory provided with `http.Dir(uploadPath)`.

Now we only need to implement the `uploadFileHandler`.

This handler will:

* Validate the maximum file size
* Validate the File and Post parameters from the request
* Check the provided file type (we only accept images and pdf)
* Create a randomized file-name
* Write the file to disk
* Handle all errors and return success, if everything works out

First, we define the handler:

```go
func uploadFileHandler() http.HandlerFunc {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
```

*Big Thanks to [Luis Villegas](https://github.com/luisguve) for an improved solution here, where we check the file size using the file header, instead of the whole request.*

Then, we parse the multipart/form-data in the request body by calling ParseMultiPartForm on the request object and passing to it the maxUploadSize constant as a parameter.

The whole request body is parsed and up to a total of the given amount of bytes of its file parts are stored in memory, with the remainder stored on disk in temporary files.
Any errors returned from this method should count as an internal server error, so we check that case and return the aproppiate error message and status code.
Errors are handled with a simple renderError helper, which returns the given error message and an according HTTP status code.

```go
if err := r.ParseMultipartForm(maxUploadSize); err != nil {
    fmt.Printf("Could not parse multipart form: %v\n", err)
    renderError(w, "CANT_PARSE_FORM", http.StatusInternalServerError)
    return
}
```

If the request could be parsed successfully, we will check and parse the form parameters `type` and `uploadFile` and read the file:

```go
fileType := r.PostFormValue("type")
file, fileheader, err := r.FormFile("uploadFile")
    if err != nil {
        renderError(w, "INVALID_FILE", http.StatusBadRequest)
        return
}
defer file.Close()
```

Then, we validate the file size comparing `maxUploadSize` and the `Size` field of the header returned by `r.FormFile`. For debugging purposes, we'll print out the file size in the console as well.
If the file size is greater than `maxUploadSize`, then we will send a `FILE_TOO_BIG` error to the client, along with a StatusBadRequest code.

```go
fileSize := fileHeader.Size
fmt.Printf("File size (bytes): %v\n", fileSize)
if fileSize > maxUploadSize {
	renderError(w, "FILE_TOO_BIG", http.StatusBadRequest)
	return
}
```

In this example, for clarity, we won't use any fanciness with the great `io.Reader` and `io.Writer` interfaces, but simply read the file into a byte array, which we will later write out again.

```go
fileBytes, err := ioutil.ReadAll(file)
if err != nil {
    renderError(w, "INVALID_FILE", http.StatusBadRequest)
    return
}
```

Now that we successfully validated and read the file, it's time to check the file type. A cheap and rather insecure way of doing this, would be to just check the file extension and trust the user didn't change it, but that just won't do for a serious application.

Gladly, the Go standard library provides us with the `http.DetectContentType` function, which only needs the first 512 bytes of the file to detect its file type based on the `mimesniff` algorithm.

```go
detectedFileType := http.DetectContentType(fileBytes)
switch detectedFileType {
case "image/jpeg", "image/jpg":
case "image/gif", "image/png":
case "application/pdf":
    break
default:
    renderError(w, "INVALID_FILE_TYPE", http.StatusBadRequest)
    return
}
```

In a real-world application, we would probably do something with the file metadata, such as saving it to a database or pushing it to an external service - in any way, we would parse and manipulate metadata. Here we create a randomized new name (this would probably be a UUID in practice) and log the future filename.

```go
fileName := randToken(12)
fileEndings, err := mime.ExtensionsByType(detectedFileType)
if err != nil {
    renderError(w, "CANT_READ_FILE_TYPE", http.StatusInternalServerError)
    return
}
newPath := filepath.Join(uploadPath, fileName+fileEndings[0])
fmt.Printf("FileType: %s, File: %s\n", detectedFileType, newPath)
```

Almost done - just one very important step left - writing the file. As mentioned above, we simply copy the byte array we got from reading the file to a newly created file handler called `newFile`.

If everything went well, we return a `SUCCESS` message to the user.

```go
newFile, err := os.Create(newPath)
if err != nil {
    renderError(w, "CANT_WRITE_FILE", http.StatusInternalServerError)
    return
}
defer newFile.Close()
if _, err := newFile.Write(fileBytes); err != nil {
    renderError(w, "CANT_WRITE_FILE", http.StatusInternalServerError)
    return
}
w.Write([]byte("SUCCESS"))
```

That's it. You can test this simple example with a dummy file-upload HTML page, cURL or a nice tool like [postman](https://www.getpostman.com/).

The full example code can be found [here](https://github.com/zupzup/golang-http-file-upload-download)

## Conclusion

This was another demonstration of how Go enables its users to write simple, yet powerful software for the web without having to deal with the countless layers of abstraction inherent in other languages and ecosystems.

In the next couple of posts, I will show several other niceties I encountered while writing my first serious web application in Go, so stay tuned. ;)

*// Edited the code after some feedback by `lstokeworth` on reddit. Thank you :)*

*// Edited the code and text after a merge request by [Luis Villegas](https://github.com/luisguve) on GitHub. Thank you :)*

#### Resources

* [Full Code Example](https://github.com/zupzup/golang-http-file-upload-download)

