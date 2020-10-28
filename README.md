# openfaas-puppeteer-template

This [OpenFaaS template](https://www.openfaas.com/) uses [docker-puppeteer by buildkite](https://github.com/buildkite/docker-puppeteer/) to give you access to [Puppeteer](https://github.com/puppeteer/puppeteer). Puppeteer is a popular tool that can automate a headless Chrome browser for scraping fully-rendered web pages.

Why do we need an OpenFaaS template? Templates provide an easy way to scaffold a microservice or function and to deploy that at scale on a Kubernetes cluster. The faasd project also gives a way for small teams to get on the experience curve, without learning anything about Kubernetes.

* Extend timeouts to whatever you want
* Run asynchronously, and in parallel
* Get a callback with the result when done
* Limit concurrency with `max_inflight` environment variable in stack.yml
* Trigger from cron, or events

## Get OpenFaaS

[Deploy OpenFaaS](https://docs.openfaas.com/deployment/) to Kubernetes, or to faasd (single-node with just containerd)

## Create a function with the template and deploy it to OpenFaaS

```bash
faas-cli template pull https://github.com/alexellis/openfaas-puppeteer-template

faas-cli new --lang puppeteer-node12 scrape-title --prefix alexellis2

faas-cli up -f scrape-title.yml
```

## Example functions and invocations

### Get the title of a webpage passed in via a JSON body

```javascript
'use strict'
const assert = require('assert')
const puppeteer = require('puppeteer')

module.exports = async (event, context) => {
  let browser
  let page
  
  browser = await puppeteer.launch({
    args: [
      // Required for Docker version of Puppeteer
      '--no-sandbox',
      '--disable-setuid-sandbox',
      // This will write shared memory files into /tmp instead of /dev/shm,
      // because Docker’s default for /dev/shm is 64MB
      '--disable-dev-shm-usage'
    ]
  })

  const browserVersion = await browser.version()
  console.log(`Started ${browserVersion}`)
  page = await browser.newPage()
  let uri = "https://inlets.dev/blog/"
  if(event.body && event.body.uri) {
    uri = event.body.uri
  }

  const response = await page.goto(uri)
  console.log("OK","for",uri,response.ok())

  let title = await page.title()
  const result = {
    "title": title
  }

  browser.close()
  return context
    .status(200)
    .succeed(result)
}
```

```bash
echo '{"uri": "https://inlets.dev/blog"}' | faas-cli invoke scrape-title \
  --header "Content-type=application/json"
```

Alternatively run async:

```bash
echo '{"uri": "https://inlets.dev/blog"}' | faas-cli invoke scrape-title \
  --async \
  --header "Content-type=application/json"
```

Run async, post the response to another service or receiver function:

```bash
echo '{"uri": "https://inlets.dev/blog"}' | faas-cli invoke scrape-title \
  --async \
  --header "Content-type=application/json" \
  --header "X-Callback-Url=https://en98kppbwx32.x.pipedream.net"
```

### Take a screenshot and return it as a binary response

```javascript
'use strict'
const assert = require('assert')
const puppeteer = require('puppeteer')
const fs = require('fs').promises

module.exports = async (event, context) => {
  let browser
  let page
  
  browser = await puppeteer.launch({
    args: [
      // Required for Docker version of Puppeteer
      '--no-sandbox',
      '--disable-setuid-sandbox',
      // This will write shared memory files into /tmp instead of /dev/shm,
      // because Docker’s default for /dev/shm is 64MB
      '--disable-dev-shm-usage'
    ]
  })

  const browserVersion = await browser.version()
  console.log(`Started ${browserVersion}`)
  page = await browser.newPage()
  let uri = "https://inlets.dev/blog/"
  if(event.body && event.body.uri) {
    uri = event.body.uri
  }

  const response = await page.goto(uri)
  console.log("OK","for",uri,response.ok())

  let title = await page.title()
  const result = {
    "title": title
  }
  await page.screenshot({ path: `/tmp/page.png` })

  let data = await fs.readFile("/tmp/page.png")

  browser.close()
  return context
    .status(200)
    .headers({"Content-type": "application/octet-stream"})
    .succeed(data)
}
```

```bash
echo '{"uri": "https://inlets.dev/blog"}' | \
faas-cli invoke screenshot-page --header "Content-type=application/json" > screenshot.png
```
