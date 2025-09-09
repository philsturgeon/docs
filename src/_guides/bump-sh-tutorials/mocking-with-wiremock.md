---
title: Mocking APIs with WireMock
authors: phil
excerpt: Create powerful API mocks to speed up client development using WireMock.
date: 2024-07-15
---

Once your beautiful API documentation is deployed with Bump.sh and people are happily using it, what's next? If you've been following the [API design-first workflow](_guides/api-basics/dev-guide-api-design-first.md) the next phase is gathering feedback on the proposed design before investing loads of time building it, and that's where mocking can help out.

Sharing the documentation is a good start, but you can get even better feedback by giving stakeholders a mock server to interact with. A mock server simulates an API, allowing stakeholders to see if all the data they need is available, and give feedback on how easily their workflows can be solved based on the endpoints in the API. This can be thought of like a study group, something user researchers are used to doing for frontends, but is just as valuable for APIs.

Historically people would create "prototypes", which were basically entirely applications coded up in a framework that was quicker to write than it was to run, but these days that feels a lot like wasting time and money when Bump.sh users already have the API described entirely by OpenAPI already. Why not just use OpenAPI-powered mock servers [WireMock Cloud](https://www.wiremock.io/) to do the hard work for you instead of building a throwaway application.

Let's take a look at using WireMock Cloud, and see how we can tie it into your existing Bump.sh documentation.

## Step 1: Install WireMock CLI

Cloud app with a CLI so grab that.

```
npm install -g @wiremock/cli

wiremock login
```

![](/images/guides/mocking-with-wiremock/register-for-wiremock-cloud.png)

Make a new Wiremock Cloud account, or login with Google or GitHub accounts.

The GitHub authorization is minimal and they're only trying to get your email address, so that's usually the easiest way to go.

![](/images/guides/mocking-with-wiremock/wiremock-github-auth.png)

Pop over to your new WireMock Cloud dashboard to continue, we'll get back to the CLI in a moment.

## Step 2: Add Your First API

In the WireMock Cloud dashboard, create a new Mock API, and when it asks to pick a type pick REST.

![](/images/guides/mocking-with-wiremock/wiremock-choose-protocol.png)

If you were using Yé Oldé Mocking Toolés this would be the part where you spend hours, days, or weeks, manually recreating the mocking interface one endpoint, request, and response at a time, hoping you got it right but not really having any good way to tell. Thankfully there's absolutely no reason to do that sort of thing in the modern OpenAPI world.

Seeing as Bump.sh users are already publishing OpenAPI documents as API documentation with CLI/CI tools, we can do the exact same thing for API mocking.

_If you are just starting out and don't have any OpenAPI documents just yet, go and grab the [Train Travel API](https://github.com/bump-sh-examples/train-travel-api) so you can follow along learning about WireMock without having to go [learn everything about OpenAPI](_guides/openapi/specification/v3.1/introduction/what-is-openapi.md) first._

```shell
wiremock push open-api <wiremock-id> --file openapi.yaml
```

The response from this command will say something like: 

```text
Successfully updated OpenAPI document for mock API with ID: <wiremock-id>
```

That means its worked nicely, so we can now go kick the tires of our new mock API.

## Step 3: Try out the Mock Endpoints

Whatever OpenAPI document you uploaded, the server URL will now be something like:

```
https://e0oo3.wiremockapi.cloud/
```

All `paths:` defined in the OpenAPI will be available with the methods they support, so in the example of the train travel API we can `GET` the `/stations` path with the following command:

```
curl --request GET https://e0oo3.wiremockapi.cloud/stations | jq .

{
  "data": [
    {
      "id": "efdbb9d1-02c2-4bc3-afb7-6788d8782b1e",
      "name": "Berlin Hauptbahnhof",
      "address": "Invalidenstraße 10557 Berlin, Germany",
      "country_code": "DE",
      "timezone": "Europe/Berlin"
    },
    {
      "id": "b2e783e1-c824-4d63-b37a-d8d698862f1d",
      "name": "Paris Gare du Nord",
      "address": "18 Rue de Dunkerque 75010 Paris, France",
      "country_code": "FR",
      "timezone": "Europe/Paris"
    }
  ],
  "links": {
    "self": "https://api.example.com/stations&page=2",
    "next": "https://api.example.com/stations?page=3",
    "prev": "https://api.example.com/stations?page=1"
  }
}
```

The default behavior of WireMock is to generate the responses from the `examples` defined in the OpenAPI document. 

This is a great way to get started, but you can do so much more with WireMock if you'd like to provide a whole lot more /stations` than could reasonably be fit into the examples showing up on the API Documentation.

## Step 4: Adding Data Sources

WireMock has a feature called data sources, which let you load a whole bunch more data into the mock server than it can glean from examples alone. 

Head over to the "Data Sources" tab on the top navigation, then click the big "Create new data source" button. 
The WireMock Cloud interface suggests you can in fact connect any old sort of database source, but you'll need to talk to the team if you want to do anything other than a CSV for now. Fair enough, that sounds complicated. Let's keep it easy for now and stick to CSV. 

The best way to do this right now is to pop a bunch of data in there (doesn't have to be everything ever, just a bit more than you would want to pop into an OpenAPI example), so lets shove a few hundred examples into a CSV file with names that match the properties expected in our OpenAPI. In this example of a `stations.csv` we have given the data source a name of `stations`.

![](/images/guides/mocking-with-wiremock/create-new-data-source.png)

Once that data source exists we need to hook up one of our responses to a data source, so the mocking engine knows it can use that. Pop back to Mock APIs, search for the `stations` in this example, then pick the new data source for that response.

![](/images/guides/mocking-with-wiremock/hook-up-data-source.png)


Cant just magically produce JSON so we gotta make a template, because how would it know if theres a data [] or some other sort of envelope or wrapper. 

https://docs.wiremock.io/data-sources/overview#rendering-data-in-response


```json
{{#formatJson}}
{
  "data": [
    {{~#arrayJoin ',' data.items[0,1] as |item|~}}
      {
        "id": {{item.id}},
        "name": "{{item.name}}"
      }
    {{/arrayJoin}}
  ]
}
{{/formatJson}}
```


Want to make pagination work? 

```
{{#formatJson}}
{{val request.query.page or='1' assign='page'}}
{{#assign 'limitParameter'}}{{#if request.query.limit}}&limit={{request.query.limit}}{{/if}}{{/assign}}
{
  "data": [
    {{~#arrayJoin ',' data.items as |station|~}}
    {
      "id": "{{station.id}}",
      "name": "{{station.name}}",
      "address": "{{station.address}}",
      "time_zone": "{{station.timezone}}",
      "country_code": "{{station.country_code}}"
    }
    {{/arrayJoin}}
  ],
  "links": {
    "self": "{{request.baseUrl}}{{request.url}}",
    "next": "{{request.baseUrl}}/stations?page={{math page '+' 1}}{{limitParameter}}"{{#and (neq page 1) (neq page '1')}},
    "prev": "{{request.baseUrl}}/stations?page={{math page '-' 1}}{{limitParameter}}"{{/and}}
  }
}
{{/formatJson}}
```


The templating system here is handlebars, which will feel familiar to some of you as it's been around for ~15 years. If you need to learn basics (or refresh your memory) the Wiremock [Templating Basics](https://docs.wiremock.io/response-templating/basics) guide will get you there.



## Dynamic responses for writes


```
curl --request POST \
  --url https://wm-tom-train-travel.wiremockapi.cloud/bookings \
  --header 'Authorization: Bearer 123' \
  --header 'Content-Type: application/json' \
  --data '{
  "trip_id": "4f4e4e1-c824-4d63-b37a-d8d698862f1d",
  "passenger_name": "BigMouthBillyBass",
  "has_bicycle": true,
  "has_dog": false
}'
```




## Automate Mock Updates

Most Bump.sh users use some form of Continuous Integration (CircleCI, GitHub Actions, Jenkins, etc.) to push changes to their API documentation whenever the source code is changed, and you can work this way with WireMock, or there are some alternatives you can try out.

Below is the standard GitHub Action used to deploy API changes to Bump.sh with one modification to also deploy changes to your WireMock server.

```yaml
# .github/workflows/deploy-docs.yml
name: Deploy API documentation

on:
  push:
    branches:
      - main

jobs:
  deploy-openapi:
    if: ${{ github.event_name == 'push' }}
    name: Deploy API documentation on Bump.sh
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Deploy API documentation
        uses: bump-sh/github-action@v1
        with:
          doc: <your-doc-id-or-slug>
          token: ${{secrets.BUMP_TOKEN}}
          file: api/openapi.yaml

      - uses: wiremock/import-github-action@v1
        with:
          specificationFiles: 'api/openapi.yaml:true'
          wiremockURL: 'https://mocks.example.com/api/'
          keycloakClientId:  ${{ secrets.MICROCKS_SERVICE_ACCOUNT }}
          keycloakClientSecret:  ${{ secrets.MICROCKS_SERVICE_ACCOUNT_CREDENTIALS }}

```

You'll need to set up some secrets on your repository for that WireMock service account, but then you're done! A fully functioning mock server running on the cloud, which anyone can interact with internally or externally depending on how you set it up.

> Learn more about [WireMock Automation](https://wiremock.io/documentation/guides/automation/) to see how to push updates to WireMock using other CI systems, via the API, or using the CLI elsewhere. You can also use the [WireMock Scheduler](https://wiremock.io/documentation/guides/usage/importing-content/#2-import-content-via-importer) instead, to pull content from a repo on a regular schedule instead of pushing.
{: .info }

## Step 5: Add Mock Server to API Documentation

Once the mock server is up and running, you can help make it easier to find by adding it to your Bump.sh API documentation. To do this, we can add the server URL into the servers list like this:

```yaml
servers:
  - url: https://api.example.com
    description: Production
  
  - url: https://mocks.example.com/rest
    description: Mock Server
```

Adding this second server URL will offer users a dropdown menu in the Bump.sh documentation.

![The select box appears on the Bump.sh API documentation allowing users to pick between servers based on server name](TODO)

The mock server is now an option, and all of the URLs and example HTTP requests will show up using the chosen server URL.

![A screenshot of the API documentation updated to contain the mocks.example.com after mock server has been selected](TODO)

If there's no production API only the mock server is ready then only define that:

```yaml
servers:
  - url: https://mocks.example.com/rest
    description: Mock Server
```

> Perhaps you cannot update the OpenAPI document to get that server added as it's generated by somebody else or hosted online, in which case take a look at our [Overlays guide](_guides/openapi/augmenting-generated-openapi.md).
{: .info }
