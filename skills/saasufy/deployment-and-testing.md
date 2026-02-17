# Saasufy Deployment and Testing

## Testing

For testing, the user should be able to open the relevant `.html` file directly by double clicking on it (`file://` protocol). If they run into trouble, you can suggest using a HTTP server - Please offer to install one for them.

## Deploying the Service

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -XPOST 'https://saasufy.com/api/service/start'
```

The service can be deployed manually by logging into the `saasufy.com` Dashboard and clicking on the `Deploy service` button. It's critical to deploy the service after schema changes in order for them to take effect.

## Stopping the Service

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -XPOST 'https://saasufy.com/api/service/stop'
```

The service can be stopped manually by logging into the `saasufy.com` Dashboard and clicking on the `Stop service` button.