# cloud2

## Gitea on Katacoda

https://katacoda.com/courses/openshift/playground

```
oc new-project mygitea
oc new-app -f https://github.com/pschiffe/cloud2/raw/master/gitea-template.yml
oc get all
```

Wait until:
```
NAME               READY     STATUS    RESTARTS   AGE
po/gitea-1-ln2jn   1/1       Running   0          44s
```

Get url:
```
NAME           HOST/PORT                                                         PATH      SERVICES   PORT      TERMINATION   WILDCARD
routes/gitea   gitea-myproject.2886795373-80-simba02.environments.katacoda.com             gitea      <all>                   None
```

Install Gitea with following options:

**Database Type**: SQLite3  
**Domain**: gitea-myproject.2886795373-80-simba02.environments.katacoda.com  
**Application URL**: http://gitea-myproject.2886795373-80-simba02.environments.katacoda.com/  

Create user in **Admin Account Settings**.

Click **Install Gitea**

After the installation is done, you should be logged in your Gitea instance. If you see HTTPS error, accept self-signed certificate, or use HTTP version of the site.

## Import example app

In top right corner click on the plus icon and choose **New Migration**. Fill in:

**Clone Address**: https://github.com/pschiffe/ocp-go-webserver.git  
**Repository Name**: ocp-go-webserver  

## Deploy example app in OpenShift

Copy raw url of file `openshift/templates/go-web.yml`. In OpenShift on Katacoda, create new project and deploy your example app:

```
oc new-project myapp
oc new-app -f https://github.com/pschiffe/ocp-go-webserver-play/raw/master/openshift/templates/go-web.yml --param=SOURCE_REPOSITORY_URL=https://github.com/pschiffe/ocp-go-webserver-play.git
oc get all
```
Wait until:
```
NAME                READY     STATUS      RESTARTS   AGE
po/go-web-1-939bh   1/1       Running     0          5m
```

Get url and test it in the browser:
```
NAME            HOST/PORT                                                      PATH      SERVICES   PORT      TERMINATION   WILDCARD
routes/go-web   go-web-myapp.2886795306-80-simba02.environments.katacoda.com             go-web     <all>                   None
```

You should see `Welcome to my website!`

## Scale app to 5 instances

```
oc scale dc/go-web --replicas=5
oc get all
```

Observe.

## Configure build web hook

In OpenShift, get url of webhook:
```
oc describe bc/go-web
```

```
Webhook Generic:
  URL:  https://172.17.0.42:8443/oapi/v1/namespaces/myapp/buildconfigs/go-web/webhooks/5dRNoPIGPVRkbHo3YHKGexnKORCSaKUd8C4mfrNu/generic
```

Replace `172.17.0.42:8443` with url of your Dashboard url from left pane, something like `2886795306-8443-simba02.environments.katacoda.com`, so the proper webhook is `https://2886795306-8443-simba02.environments.katacoda.com/oapi/v1/namespaces/myapp/buildconfigs/go-web/webhooks/5dRNoPIGPVRkbHo3YHKGexnKORCSaKUd8C4mfrNu/generic`.

Now copy this webhook to the repo settings in your Gitea. There is `Settings` button in upper right area under the `Fork` button, then `Webhooks` and `Add Webhook` -> `Gitea`. Fill in:

**Payload URL**: https://2886795306-8443-simba02.environments.katacoda.com/oapi/v1/namespaces/myapp/buildconfigs/go-web/webhooks/5dRNoPIGPVRkbHo3YHKGexnKORCSaKUd8C4mfrNu/generic  
**When should this webhook be triggered?**: Just the push event  

Click `Add Webhook`.

## Make a change to the app

In Gitea edit `main.go` file, change string on line `fmt.Fprintf(w, "Welcome to my website!")` to something different and save changes. With `oc get all` you can see now, that new build was started and in a moment, new version of your app will be deployed.

## Deployment Config: Recreate Strategy

```yaml
strategy:
  type: Recreate
```

https://docs.openshift.org/latest/dev_guide/deployments/deployment_strategies.html#recreate-strategy

## Deployment Config: Rolling Strategy

```
oc edit dc/go-web
```

Vim will be opened by default, if you don't know how to use it, run:

```
EDITOR=nano oc edit dc/go-web
```

```yaml
strategy:
  type: Rolling
  rollingParams:
    updatePeriodSeconds: 15
    timeoutSeconds: 60
```

https://docs.openshift.org/latest/dev_guide/deployments/deployment_strategies.html#rolling-strategy

### Break the code and see what happens

Instead of:
```go
  http.HandleFunc("/", func (w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Welcome to my website!"))
  })
```

Try:
```go
  http.HandleFunc("/", func (w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusInternalServerError)
    w.Write([]byte("500 - Something bad happened!"))
  })
```

### And now without canary

```
oc edit dc/go-web
```

And delete following sections:

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 8080
    scheme: HTTP
  initialDelaySeconds: 10
  timeoutSeconds: 1
  periodSeconds: 10
  successThreshold: 1
```

and

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 8080
    scheme: HTTP
  initialDelaySeconds: 5
  timeoutSeconds: 1
  periodSeconds: 10
  successThreshold: 1
  failureThreshold: 3
```

Observe.. and when you have enough:

```
oc rollout undo --to-revision=2 dc/go-web
```

# Plan B

If Katacoda is breaking too much for you, fork my repo ocp-go-webserver on GitHub: https://github.com/pschiffe/ocp-go-webserver and use that as a repository instead of Gitea. You can configure webhook as well the same way as described above.

# Plan C

This section doesn't need git or source or anything, just demonstrates advanced deployment strategies.

## Blue - Green Deployment

```
oc new-project blue-green
```

```
oc new-app openshift/deployment-example:v1 --name=example-green
oc new-app openshift/deployment-example:v2 --name=example-blue
```

```
oc expose svc/example-green --name=bluegreen-example
```

```
oc edit route/bluegreen-example
```

```yaml
to:
  name: example-blue
```

https://docs.openshift.org/latest/dev_guide/deployments/advanced_deployment_strategies.html#advanced-deployment-strategies-blue-green-deployments

## A/B Deployment

```
oc new-project ab
```

```
oc new-app openshift/deployment-example:v1 --name=ab-example-a
oc new-app openshift/deployment-example:v2 --name=ab-example-b
```

```
oc expose svc/ab-example-a --name=ab-example
```

```
oc set route-backends ab-example ab-example-a=66 ab-example-b=33
```

```
oc annotate --overwrite routes/ab-example haproxy.router.openshift.io/disable_cookies=true
```

https://docs.openshift.org/latest/dev_guide/deployments/advanced_deployment_strategies.html#advanced-deployment-a-b-deployment
