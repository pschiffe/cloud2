# cloud2

## Gitea on Katacoda

[https://katacoda.com/courses/openshift/playground](https://katacoda.com/courses/openshift/playground)

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
oc new-app -f http://gitea-mygitea.2886795306-80-simba02.environments.katacoda.com/myuser/ocp-go-webserver/raw/branch/master/openshift/templates/go-web.yml --param=SOURCE_REPOSITORY_URL=http://gitea-mygitea.2886795306-80-simba02.environments.katacoda.com/myuser/ocp-go-webserver.git
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

## Made a change to the app

In Gitea edit `main.go` file, change string on line `fmt.Fprintf(w, "Welcome to my website!")` to something different and save changes. With `oc get all` you can see now, that new build was started and in a moment, new version of your app will be deployed.
