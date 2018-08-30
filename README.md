# Push Prometheus Pushgateway container into OCP

![gluster-gauges](https://github.com/ikke-t/gluster-pv-monitor-hack/raw/master/gluster-grafana.jpg)

In case you can connect to docker.io from OCP, you can also use the import command directly. Make sure the image get's to corresponding namespace which you have in DC below.

```sh
oc login
skopeo copy --dest-creds `oc whoami`:`oc whoami -t` docker://docker.io/prom/pushgateway:latest docker://docker-registry.apps.foobar.com:443/openshift/prometheus-pushgateway
```

# Enable prometheus pushgateway in OCP

## Add deploymentconfig for Prometheus pushgateway pod

```yaml
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  labels:
    app: pushgateway
  name: pushgateway
  namespace: openshift-metrics
spec:
  replicas: 1
  selector:
    app: pushgateway
    deploymentconfig: pushgateway
  strategy:
    activeDeadlineSeconds: 21600
    rollingParams:
      intervalSeconds: 1
      maxSurge: 25%
      maxUnavailable: 25%
      timeoutSeconds: 600
      updatePeriodSeconds: 1
    type: Rolling
  template:
    metadata:
      labels:
        app: pushgateway
        deploymentconfig: pushgateway
    spec:
      containers:
        - image: prom/pushgateway
          imagePullPolicy: Always
          name: pushgateway
          ports:
            - containerPort: 9091
              protocol: TCP
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
  test: false
  triggers:
    - type: ConfigChange
    - imageChangeParams:
        automatic: true
        containerNames:
          - pushgateway
        from:
          kind: ImageStreamTag
          name: 'pushgateway:latest'
          namespace: openshift-metrics
      type: ImageChange
```

## Creae service for pushgateway

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: pushgateway
  name: pushgateway
  namespace: openshift-metrics
spec:
  ports:
    - name: 9091-tcp
      port: 9091
      protocol: TCP
      targetPort: 9091
  selector:
    deploymentconfig: pushgateway
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

## Add Route for pushgateway

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: pushgateway
  name: pushgateway
  namespace: openshift-metrics
spec:
  host: pushgateway-openshift-metrics.apps.openshift.foobar.com
  port:
    targetPort: 9091-tcp
  to:
    kind: Service
    name: pushgateway
    weight: 100
  wildcardPolicy: None
  ```

# Enable data collection and push in Gluster node

We set this up only on one host, as all had identical replicas. Until Gluster somehow screwed up and started having orphan stuff.

## Add script to query gluster PV usage

```sh
#!/bin/bash
# Report GB free for each non-OS VG
touch /tmp/pvs_mon.last
/sbin/pvs | grep -v rhel | tail -n +2 | awk -v host="$(hostname -s)" '{ print host"_"$2" "$6 }' | sed 's/<//' | sed 's/-/_/' | sed 's/g$//' | while read line; do echo $line | curl --data-binary @- http://pushgateway-openshift-metrics.apps.foobar.com/metrics/job/pvsmon/pvs/vg; done
```

## Add crontab to run the PV query string periodicly. This will push the data to prometheus exporter

```
#Ansible: Gluster VG Metric Push
*/10 * * * * /root/pvs_mon.sh
```

## Grafana

We lost this part, due there was no PV for it :( need to redo, but it was yeasy piecy.
  
