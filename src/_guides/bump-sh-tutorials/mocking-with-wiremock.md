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
```

![]()

## Step 2: Add Your First API

There are several ways to get OpenAPI into WireMock, but for the sake of simplicity we're going to use the web interface to upload your OpenAPI. If you are just starting out and don't have any OpenAPI yet, why not use the [Train Travel API](https://github.com/bump-sh-examples/train-travel-api) for now. 

```
wiremock push open-api e0oo3 --file train-travel-api/openapi.yaml
```



## Step 3: Try out the Mock Endpoints


copy the mock base url 

```
https://e0oo3.wiremockapi.cloud/
```



Quickly try out the mock server using curl, or your favourite HTTP client. 

This grabs the list of stations from the mock server, which is a collection endpoint that returns a list of stations in the Train Travel API.

```
curl --request GET https://e0oo3.wiremockapi.cloud/stations | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   512  100   512    0     0   1160      0 --:--:-- --:--:-- --:--:--  1158
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

This is generated from examples in the OpenAPI document but the data is not real, it's just sample data. This is a great way to get started, but you can do so much more with Wiremock using its "data sources".



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



Getting hateoas links to work.


Finally, that HATEOAS link at the bottom there is not right. The id is `a248a638-000a-44b4-b19b-0ca30507a940`, so how can we get that showing up in the link? If we used `{{ uuid() }}` again it would be generate a second different UUID, WireMock templating has a brilliant feature called capture.

```yaml
examples:
  new_booking:
    summary: New Booking
    value: |-
      {
        "id": "{{ uuid() > put(bookingId) }}",
        "trip_id": "{{ request.body/trip_id }}",
        "passenger_name": "{{ request.body/passenger_name }}",
        "has_bicycle": {{ request.body/has_bicycle }},
        "has_dog": {{ request.body/has_dog }},
        "links": {
          "self": "https://api.example.com/bookings/{{ bookingId }}"
        }
      }
```

Upload that again. Run the command again. 

```yaml
{
  "id": "b4441c3d-b659-4f57-95d3-ce6ee592da7c",
  "trip_id": "4f4e4e1-c824-4d63-b37a-d8d698862f1d",
  "passenger_name": "New Passenger",
  "has_bicycle": false,
  "has_dog": false,
  "links": {
    "self": "https://api.example.com/bookings/b4441c3d-b659-4f57-95d3-ce6ee592da7c"
  }
}
```

Perfect! Now you can do almost anything you need to do with the API.

## Step 4: Automate Mock Updates

Most Bump.sh users use some form of Continuous Integration (CircleCI, GitHub Actions, Jenkins, etc.) to push changes to their API documentation whenever the source code is changed, and you can work this way with WireMock, or there are some alternatives you can try out.

### Update Mocks with Continuous Integration

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

      - uses: microcks/import-github-action@v1
        with:
          specificationFiles: 'api/openapi.yaml:true'
          microcksURL: 'https://mocks.example.com/api/'
          keycloakClientId:  ${{ secrets.MICROCKS_SERVICE_ACCOUNT }}
          keycloakClientSecret:  ${{ secrets.MICROCKS_SERVICE_ACCOUNT_CREDENTIALS }}
```

You'll need to set up some secrets on your repository for that WireMock service account, but then you're done! A fully functioning mock server running on the cloud, which you can interact with internally or externally depending on how you set it up.

> Learn more about [WireMock Automation](https://microcks.io/documentation/guides/automation/) to see how to push updates to WireMock using other CI systems, via the API, or using the CLI elsewhere. You can also use the [WireMock Scheduler](https://microcks.io/documentation/guides/usage/importing-content/#2-import-content-via-importer) instead, to pull content from a repo on a regular schedule instead of pushing.
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

![The select box apears on the Bump.sh API documentation allowing users to pick between servers based on server name](/images/guides/mocking-with-microcks/multiple-servers.png)

The mock server is now an option, and all of the URLs and example HTTP requests will show up using the chosen server URL.

![A screenshot of the API documentation updated to contain the mocks.example.com after mock server has been selected](/images/guides/mocking-with-microcks/bump-mock-server-curl.png)

If there's no production API only the mock server is ready then only define that:

```yaml
servers:
  - url: https://mocks.example.com/rest
    description: Mock Server
```

> Perhaps you cannot update the OpenAPI document to get that server added as it's generated by somebody else or hosted online, in which case take a look at our [Overlays guide](_guides/openapi/augmenting-generated-openapi.md).
{: .info }
