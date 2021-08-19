# Using API

_Using API to programmatically access Gen3 services' endpoints_

**Required:** You will need [curl](https://curl.se/) and [jq](https://stedolan.github.io/jq/). On macOS, you can install like so: `brew install curl jq`

### Getting Bearer Token

- First, download **API Key** by visiting to your [Profile](https://gen3.cloud.dev.umccr.org/identity) > Create API key > download `credentials.json`
- Next, acquire short-lived (1 hour) bearer token from Fence service as follows:
```
cd $HOME
mkdir -p "$HOME/.gen3/"

curl -s -X POST https://gen3.cloud.dev.umccr.org/user/credentials/api/access_token \
  -d @"$HOME/.gen3/credentials.json" -H "Content-Type: application/json" -o gen3_token.json
```

- Export token as environment variable
```
export GEN3_TOKEN=$(jq -r '.access_token' ./gen3_token.json)
env | grep GEN3_TOKEN
```

### Token Details

- If you'd like to check JWT token details with R, try `gen3_decode.R` [script](gen3_decode.R).
```
Rscript gen3_decode.R
```

- This token decoder may be useful when working with GA4GH Passport and Visa token integration for data access.

### Index API

- Get by GUID
```
curl -s https://gen3.cloud.dev.umccr.org/index/f5c4e255-2bbf-4026-9fbd-4dec3ae5f886 | jq
```

### DRS API

_[GA4GH Data Repository Service API](https://ga4gh.github.io/data-repository-service-schemas/preview/release/drs-1.0.0/docs/)_

- Get DRS Object
```
curl -s "https://gen3.cloud.dev.umccr.org/ga4gh/drs/v1/objects/cf778361-6f6f-4128-8673-93ca69111c93" | jq
```

- Get (PreSigned) URL for fetching DRS Object bytes
```
curl -s -H "Authorization: Bearer $GEN3_TOKEN" "https://gen3.cloud.dev.umccr.org/ga4gh/drs/v1/objects/cf778361-6f6f-4128-8673-93ca69111c93/access/s3" | jq
```


## OpenAPI Documentation

- [Indexd API](https://petstore.swagger.io/?url=https://raw.githubusercontent.com/uc-cdis/Indexd/master/openapis/swagger.yaml)
- [Fence API](https://petstore.swagger.io/?url=https://raw.githubusercontent.com/uc-cdis/fence/master/openapis/swagger.yaml)


## REF

- https://gen3.org/resources/user/using-api/
- https://github.com/ga4gh/fasp-scripts
