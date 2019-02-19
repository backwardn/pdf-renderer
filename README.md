# PDF-RENDERER

This service runs in conjunction with a headless chrome browser. It listens for requests on port 9766. Upon receiving a render request, it will go to a URL, render the page and respond with a zip file. The zip file will contain a json file and a pdf file. The json file will be a list of network requests made by the page that was rendered. The pdf file will be a print-out of the page. 

## Features:

#### Network Request Polling
The service will wait until no requests are being made before printing the page.

#### QoS Reporting
In order to provide some insight into how well the page was rendered, the zip file generated by this service includes information about every network response and the status code.

#### Custom Header Passthrough
You can provide custom headers that will be used by headless chrome when making all network requests.

#### Request Correlation
If your request timed out, this will allow you to make a request again with the same correlation id and possibly get a stored version of the zip containing the pdf instead of having to generate it again. The stored version is encrypted at rest and will be eligible for deletion by the service after some time.

## Configuration

You can configure the app via environment variables.

#### DEBUG
Default Value: (bool) `false`  
Description: when "true", outputs debug log messages and enables other debugging features (ie memory usage logging)

#### PDF_RENDERER_KEY
Default Value: (string) `JKNV29t8yYEy21TO0UzvDsX2KgiWrOVy`  
Description: defines the key used to encrypt the files while at rest

#### PDF_RENDERER_WEB_SERVER_PORT
Default Value: (int) `9766`
Description: the port that the web server will listen on

#### PDF_RENDERER_STORAGE_STRATEGY
Default Value: (string) `memory`  
Description: defines the strategy by which to store a copy of the generated files. Persisted files will be overwritten if the same filename is used. Valid strategies:
* memory: the file remains in memory only
* disk: stores a copy of the encrypted file to disk, the file path can be configured via PDF_RENDERER_STORAGE_DIRECTORY
* s3: stores a copy of zip to s3, the bucket can be configured with environment variable PDF_RENDERER_S3_BUCKET. Must have a valid AWS credentials file in home dir (https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

#### PDF_RENDERER_STORAGE_DIRECTORY
Default Value: (string) `/tmp/pdf-renderer/`  
Description: defines the directory in which the encrypted zip files are stored when using the "disk" storage strategy

#### PDF_RENDERER_CORRELATION_STORAGE_DIRECTORY
Default Value: (string) `/tmp/pdf-renderer-correlation/`  
Description: defines the directory in which the encrypted zip files are temporarily stored for correlation purposes

#### PDF_RENDERER_CORRELATION_RETENTION_DURATION
Default Value: (string) `1h`  
Description: defines the minimum age at which a temporarily stored correlation file becomes eligible for deletion

#### PDF_RENDERER_REQUEST_POLL_RETRIES
Default Value: (int) `10`  
Description: defines the number of times to poll the browser for new network requests/responses when no requests are pending before assuming the page is done rendering

#### PDF_RENDERER_S3_BUCKET 
Default Value: **NOT SET**  
Description: defines the bucket in which the zip is stored if storage strategy is set to 's3'. Note: This must be a globally unique name.

#### PDF_RENDERER_REQUEST_POLL_INTERVAL
Default Value: (time.Duration) `1s`  
Description: defines the amount of time between each poll for new network requests/responses

#### PDF_RENDERER_PRINT_DEADLINE
Default Value: (time.Duration) `5m`  
Description: defines the maximum amount of time to spend on any given render request before simply printing whatever is there 

## Usage
```
cd $GOPATH
go get github.com/rapid7/pdf-renderer
cd src/github.com/rapid7/pdf-renderer
go build
./pdf-renderer
```

```
curl -X POST \
  http://localhost:9766/render \
  -H 'Content-Type: application/json' \
  -d '{
	"correlationId": "7667e2de-2f21-4ab7-9afb-402de2a6468f",
	"fileName": "7667e2de-2f21-4ab7-9afb-402de2a6468f.zip",
	"targetUrl": "https://example.com",
	"headers": {
		"Cookie": "cookiename: cookievalue"
	},
	"orientation": "Portrait",
	"printBackground": true,
	"marginTop": 0.4,
	"marginRight": 0.4,
	"marginBottom": 0.4,
	"marginLeft": 0.4
}'
```

The response is a zip file with the following contents:
```
<correlationId>.json
<correlationId>.pdf
```

The json file structured as such:
```
[
    {
        "url": "http://example.com",
        "status": "<status code>",
        "statusText": "<status text>"
    }, ...
]
```

## Special Notes
* Please be aware that the correlationId you provide is used to name the file that's temporarily stored onto the file system. This means that if the correlation id is a relative path, you could accidentally write to sections of the file system that you did not intend to write to.
* Please be cautious with the potential values for target url and the potential recipients of the response. If the value of the targetUrl is a sensitive url, sensitive data could be exposed. For example, setting it to 169.254.169.254 (aws meta-data service) could expose information that you might not want exposed.

## TODO
* create a unique bucket for s3 tests and delete it afterwards (blocker for next release)
* add support for other headless browsers
* make the QoS more robust (request method, time to generate, etc)
* blacklist for targetUrl values
* enable/disable for various features
* add a way to return only the json (correlation is problematic here)
* use the ec2 metadata service to resolve the region

