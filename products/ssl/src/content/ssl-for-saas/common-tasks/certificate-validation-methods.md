---
order: 3
pcx-content-type: interim
---

# Validate certificates

## HTTP (Recommended)

By far the easiest method for validating hostnames and issuing certificates is using “HTTP” validation. With this method, the only thing that your end-customers need to do is add a CNAME to your `$CNAME_TARGET` and Cloudflare will take care of the rest.

When you POST to the custom_hostnames endpoint and specify `http` as the validation method, Cloudflare will contact one of our Certificate Authority providers and indicate that we’d like to issue certificates for the specified hostname. The CA will then inform Cloudflare that we need to “demonstrate control” of this hostname by returning a `$DCV_TOKEN` at a specified `$DCV_FILENAME`; both the token and the filename are randomly generated by the CA and not known to Cloudflare ahead of time.

For example, if you create a new custom hostname for `site.example.com`, the CA might ask us to return the value `ca3-38734555d85e4421beb4a3e6d1645fe6` for a request to `http://site.example.com/.well-known/pki-validation/ca3-39f423f095be4983922ca0365308612d.txt"`. As soon as we receive that value from the CA we make it accessible at our edge and ask the CA to confirm it’s there so that they can complete validation and the certificate order.

<Aside type='note' header='Note'>

Cloudflare is able to serve the random token shown above from our edge due to the fact that site.example.com has a CNAME in place to `$CNAME_TARGET`, which ultimately resolves to Cloudflare IPs. If your customer has not yet added the CNAME, the attempt by the CA to retrieve that token will fail and the process will not complete.

We will attempt to retry this validation check for a finite period before timing out; see the <a href="/ssl-for-saas/validation-backoff-schedule">Validation Retry Schedule</a> for more details.

</Aside>

If you would like to complete the issuance process before asking your customer to update their CNAME (or before changing the resolution of your target CNAME to be proxied by Cloudflare), the domain control validation token can be served from *your* origin server, as described below.

--------

## Other

For your customers that already have HTTPS on their custom domain, e.g., they’re self-hosted or you’ve manually provisioned, and cannot tolerate a couple minutes of downtime during cutover, additional “pre-validation” methods are available.

The validation methods below can be used to obtain a certificate in advance of setting the CNAME to your `$CNAME_TARGET`.

### TXT record

This validation method allows your customer to add a TXT record to prove control of their hostname. To do so you should create a new custom hostname using `"method":"txt"` (rather than `"method":"http"`).

```bash
$ curl -sX POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/custom_hostnames" \
       -H "X-Auth-Email: {email}" -H "X-Auth-Key: {key}" \
       -H "Content-Type: application/json" \
       -d '{"hostname":"another.example.com", "ssl":{"method":"txt","type":"dv"}}'

{
  "result": {
    "id": "46f8849a-72c9-49e0-9e42-857297d89306",
    "hostname": "another.example.com",
    "ssl": {
      "id": "639e731e-ab78-4162-badf-4563708525bb",
      "type": "dv",
      "method": "txt",
      "status": "initializing"
    }
  },
  "success": true
}
```

After a few seconds, i.e., once the state has transition from `initializing` to `pending_validation`, request the status and you’ll see a payload similar to the following:

```bash
$ curl -sX GET "https://api.cloudflare.com/client/v4/zones/{zone_id}/custom_hostnames/46f8849a-72c9-49e0-9e42-857297d89306" \
       -H "X-Auth-Email: {email}" -H "X-Auth-Key: {key}"
{
  "result": {
    "id": "46f8849a-72c9-49e0-9e42-857297d89306",
    "hostname": "another.example.com",
    "ssl": {
      "id": "639e731e-ab78-4162-badf-4563708525bb",
      "type": "dv",
      "method": "txt",
      "status": "pending_validation",
      "txt_name": "another.example.com",
      "txt_value": "ca3-e95fb6a33c3648daba3b05faf3d79410",
    }
  },
  "success": true
}
```

You should then ask your customer to create a TXT record with name `another.example.com` and content `ca3-e95fb6a33c3648daba3b05faf3d79410` at their authoritative DNS provider. Once this TXT is in place, validation and certificate issuance will automatically complete.

If you’d like to request an immediate recheck, [rather than wait for the next retry](/ssl-for-saas/validation-backoff-schedule/), you can simply send a `PATCH` as follows:

```bash
$ curl -X PATCH "https://api.cloudflare.com/client/v4/zones/{zone_id}/custom_hostnames/46f8849a-72c9-49e0-9e42-857297d89306" \
       -H "X-Auth-Email: {email}" -H "X-Auth-Key: {key}" \
       -H "Content-Type: application/json" \
       -d '{"hostname":"another.example.com", "ssl":{"method":"txt","type":"dv"}}'
```

### Email

Email based validation will send an approval email to the contacts listed for a given domain in WHOIS, along with the following addresses: `admin@`, `administrator@`, `hostmaster@`, `postmaster@`, and `webmaster@`.

First, create a new hostname using `"method":"email"`:

```bash
$ curl -sX POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/custom_hostnames" \
       -H "X-Auth-Email: {email}" -H "X-Auth-Key: {key}" \
       -H "Content-Type: application/json" \
       -d '{"hostname":"emailval.example.com", "ssl":{"method":"email","type":"dv"}}'

{
  "result": {
    "id": "16798830-42c1-4e4f-82b4-4695ee8b62e4",
    "hostname": "emailval.example.com",
    "ssl": {
      "id": "086707a7-d3cc-44e4-b483-5c6e7e3f053e",
      "type": "dv",
      "method": "email",
      "status": "initializing"
    }
  },
  "success": true
}
```

Then, request the status to see the email addresses to which the approval email was sent. Note that any user that receives the email can complete the validation by following the instructions contained in the message body.

```bash
$ curl -sX GET "https://api.cloudflare.com/client/v4/zones/{zone_id}/custom_hostnames/16798830-42c1-4e4f-82b4-4695ee8b62e4" \
       -H "X-Auth-Email: {email}" -H "X-Auth-Key: {key}"
{
  "result": {
    "id": "16798830-42c1-4e4f-82b4-4695ee8b62e4",
    "hostname": "emailval.example.com",
    "ssl": {
      "id": "086707a7-d3cc-44e4-b483-5c6e7e3f053e",
      "type": "dv",
      "method": "email",
      "status": "pending_validation",
      "emails": [
        "admin@example.com",
        "administrator@example.com",
        "hostmaster@example.com",
        "postmaster@example.com",
        "webmaster@example.com",
        "7biatxra499y@contactprivacy.email"
      ]
    }
  },
  "success": true
}
```

The addresses listed above will receive an email from `support@certvalidate.cloudflare.com` that looks like this:

![Certificate Validation Email](../static/certvalidate-email.png)

They can either click on the “Review Certificate Request” button or on the https://certvalidate.cloudflare.com hyperlink. Doing so will bring your customer to a page like the one shown below. Here they should check the ”I approve..” box and then click the “Approve Certificate” button.

![Certificate Validation Approval](../static/certvalidate-approve.png)

As soon as the domain owner has clicked the link in this email and clicked “Approve” on the validation page, the certificate will automatically transition from Pending Validation to Pending Issuance to Pending Deployment and finally Active.

### HTTP (manual)

If you would like to serve the DCV tokens described above from your own origin, you can follow the instructions below. This technique is typically used by organizations that already have a large deployed base of custom domains with HTTPS support. By serving the token from your origin, you can complete validation and issuance *before* proxying your `$CNAME_TARGET` and domain through Cloudflare.

First, make a request using `"method":"http"`:

```bash
$ curl -sXPOST "https://api.cloudflare.com/client/v4/zones/{zone_id}/custom_hostnames" \
       -H "X-Auth-Email: {email}" -H "X-Auth-Key: {key}" \
       -H "Content-Type: application/json" \
       -d '{"hostname":"http-preval.example.com", "ssl":{"method":"http","type":"dv"}}'
{
  "result": {
    "id": "3aa0e60f-7622-47a4-8519-7a5fd7eb7145",
    "hostname": "http-preval.example.com",
    "ssl": {
      "id": "caacdcb1-4d61-4efe-a7a5-0dd584aa4d29",
      "type": "dv",
      "method": "http",
      "status": "initializing"
    }
  },
  "success": true
}
```

Next, wait a few seconds for the status to transition from `initializing` to `pending_validation`, the step that obtains the random path and token from the CA, and then request status:

```bash
$ curl -sXGET -H "X-Auth-Key: $MYAPIKEY" -H "X-Auth-Email: $MYEMAIL" https://api.cloudflare.com/client/v4/zones/$MYZONETAG/custom_hostnames/3aa0e60f-7622-47a4-8519-7a5fd7eb7145
{
  "result": {
    "id": "3aa0e60f-7622-47a4-8519-7a5fd7eb7145",
    "hostname": "http-preval.example.com",
    "ssl": {
      "id": "caacdcb1-4d61-4efe-a7a5-0dd584aa4d29",
      "type": "dv",
      "method": "http",
      "status": "pending_validation",
      "http_url": "http://http-preval.example.com/.well-known/pki-validation/ca3-0052344e54074d9693e89e27486692d6.txt",
      "http_body": "ca3-be794c5f757b468eba805d1a705e44f6"
    }
  },
  "success": true
}
```

You now need to make this token at the path specified in `http_url`. This path should be publicly accessible to anyone on the internet otherwise the CA will not be able to successfully complete validation.

Here is an example NGINX configuration that will return the above token:

```txt
location "/.well-known/pki-validation/ca3-0052344e54074d9693e89e27486692d6.txt" {
        return 200 "ca3-be794c5f757b468eba805d1a705e44f6\n";
}
```

Once that configuration is live, or something like it, test that the DCV text file is in place with `curl`:

```bash
$ curl "http://http-preval.example.com/.well-known/pki-validation/ca3-0052344e54074d9693e89e27486692d6.txt"
ca3-be794c5f757b468eba805d1a705e44f6
```

On the next check cycle, Cloudflare will ask the CA to recheck the URL, complete validation, and issue the certificate. If you’d like to check immediately simply send a `PATCH` with the same payload as the initial `POST`:

```bash
$ curl -sXPATCH "https://api.cloudflare.com/client/v4/zones/{zone_id}/custom_hostnames" \
       -H "X-Auth-Email: {email}" -H "X-Auth-Key: {key}" \
       -H "Content-Type: application/json" \
       -d '{"hostname":"http-preval.example.com", "ssl":{"method":"http","type":"dv"}}'
```

### 4. CNAME (manual)

The last DCV method available is via a CNAME record. First, make a request using `"method":"cname"`:

```bash
$ curl -sXPATCH "https://api.cloudflare.com/client/v4/zones/{zone_id}/custom_hostnames" \
       -H "X-Auth-Email: {email}" -H "X-Auth-Key: {key}" \
       -H "Content-Type: application/json" \
       -d '{"hostname":"cname.example.com", "ssl":{"method":"cname","type":"dv"}}'

{
  "result": {
    "id": "51027a6e-9ca2-4d66-9d46-262411936b4d",
    "hostname": "cname.example.com",
    "ssl": {
      "id": "473c09bd-9408-4b74-b051-655e9b66c86b",
      "type": "dv",
      "method": "cname",
      "status": "pending_validation",
      "cname": "_ca3-fbc2086e83a647d4822fefa68f26fc55.cname.example.com",
      "cname_target": "dcv.digicert.com",
      "bundle_method": "ubiquitous",
      "wildcard": false,
      "certificate_authority": "digicert"
    },
    "custom_metadata": null,
    "created_at": "2019-12-09T19:14:28.898621Z"
  },
  "success": true,
  "errors": [],
  "messages": []
}
```

Next, you will need to add the CNAME record that is provided in the results, ie `name: _ca3-fbc2086e83a647d4822fefa68f26fc55.cname.example.com` and `cname_target:dcv.digicert.com`, at the Authoritative DNS provider for the hostname. This CNAME record information can also be located in the Custom Hostname section of the SSL dashboard.

The certificate should validate relatively soon after its added. If you’d like to check immediately simply send a `PATCH` with the same payload.
