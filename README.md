## How To Upload K8S Pods Logs Into OCI Object Storage via Fluentd
### Requirement
  In enterprise world, we have many pods running different services. We need to store,transfer, analyze some logs to get useful information. Last time we use EFK stack to deal such requirements. Refer [the doc](https://github.com/HenryXie1/How-To-Create-EFK-Elastic-Search-FluentD-Kibana-in-Kubernetes). In addition to that, some apps need raw logs to analyze instead of Elastic index. ie we need to dig in raw site access logs to generate specific  reports. So we need to save our logs into oracle oci object storage for this purpose. This note to show we can transfer logs from K8S Pods to object storage via Fluentd

### Preparation
* Install EFK stack into the K8S cluster. Please refer [the doc](https://github.com/HenryXie1/How-To-Create-EFK-Elastic-Search-FluentD-Kibana-in-Kubernetes). Fluentd is created as daemonset in K8S ,so each worker node will have it up running to collect logs

### Implementation
* Build a new docker images with fluent plugin s3 which we can use OCI S3 compatible API to upload logs into object storage. Please refer [oci doc](https://docs.cloud.oracle.com/iaas/Content/Object/Tasks/s3compatibleapi.htm)
 * Choose one fluentd Pod. ie  foolish-antelope-fluentd-es-v68l9
 ```
 $ kubectl get po -n devops
 NAME                                               READY     STATUS    RESTARTS   AGE
 foolish-antelope-elasticsearch-logging-0           1/1       Running   4          6d
 foolish-antelope-fluentd-es-4r82h                  1/1       Running   0          4d
 foolish-antelope-fluentd-es-v68l9                  1/1       Running   0          6d
 ```
 * use kubectl get into the pod to do some updates
 ```
 $kubectl exec -it -n devops foolish-antelope-fluentd-es-v68l9 /bin/bash
 ```
 * When you we get into the pod, we install aws ruby sdk. Download aws-sdk-2.3.22.gem aws-sdk-2.3.22.gem from [github aws ruby sdk](https://github.com/aws/aws-sdk-ruby/releases/tag/v2.3.22). We don't use the latest aws ruby sdk to avoid s3_endpoint deprecation issue.Refer [link](https://docs.fluentd.org/v1.0/articles/out_s3#s3_endpoint)
 ```
 $gem install aws-sdk-2.3.22.gem aws-sdk-2.3.22.gem
 ```
 * When you we get into the pod, we install fluent-plugin-s3 version 0.8.8. We avoid the latest verion of this plugin as it needs the latest aws ruby sdk. Please refer [the plugin github link](https://github.com/fluent/fluent-plugin-s3)
 ```
 $gem install fluent-plugin-s3 -v "0.8.8"
 ```
 * Modify the fluent.conf file. Add nginx controller logs and s3 config to upload logs to OCI. Please refer config details on [fluent official doc](https://docs.fluentd.org/v1.0/articles/config-file). Test the conf file via $fluentd -v fluent.conf --dry-run . Fix all error of dry-run before we commit changes to docker images. fluent.conf Example is like

```
# This is the root config file, which only includes components of the actual configuration
# Do not collect fluentd's own logs to avoid infinite loops.
<match fluent.**>
  @type null
</match>

# config to put nginx access logs to object storage
<source>
  @type tail
  path /var/log/containers/my-nginx-nginx-ingress-controller*.log #...or where you placed your Nginx access log
  pos_file /var/log/containers/my-nginx-nginx-ingress-controller.log.pos # This is where you record file position
  tag nginx.access #fluentd tag!
  #format nginx # Do you have a custom format? You can write your own regex.
  <parse>
    @type multi_format
    <pattern>
      format json
      time_key time
      time_format %Y-%m-%dT%H:%M:%S.%NZ
    </pattern>
    <pattern>
      format /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
      time_format %Y-%m-%dT%H:%M:%S.%N%:z
    </pattern>
  </parse>
</source>

<match nginx.access>
  @type s3
  aws_key_id *****
  aws_sec_key *****
  s3_bucket IridizeStageAccessLogs
  s3_region us-ashburn-1
  s3_endpoint https://espsnonprodint.compat.objectstorage.us-ashburn-1.oraclecloud.com
  check_apikey_on_start false
  path iridizestage
  buffer_path /var/log/fluent/s3
  s3_object_key_format "%{path}/ts=%{time_slice}/hostname=#{Socket.gethostname}/accesslogs-%{index}.%{file_extension}"
  time_slice_format %Y%m%d-%H
  time_slice_wait 10m
  utc
 </match>

 @include /etc/fluent/config.d/*.conf
 ```
  * Exit the Pod and commit the changes to docker images
  ```
  docker tag k8s.gcr.io/fluentd-elasticsearch:v2.0.4 k8s.gcr.io/fluentd-elasticsearch:backup    ---> backup old images
  docker ps |grep fluent  --> find pod NAME
  docker commit k8s_fluentd-es_foolish-antelope-fluentd-es-mc5vr_devops_6e8dffa7-08b9-11e9-986b-0a580aeda22d_0  k8s.gcr.io/fluentd-elasticsearch:v2.0.4
  ```  
  * Delete the pod , let it restart ,check the logs for any error
  ```
  kubectl delete po -n devops <pod name>
  kubectl logs -n devsops <new pod name>
  ```
  * check OCI object storage console to see some logs are uploaded. Example is like
  ```
  iridizestage/ts=20181230-09/hostname=foolish-antelope-fluentd-es-dvhkn/accesslogs-0.gz
  ```
### Conclusion
This note is to demonstrate how we can leverage fluentd and its s3 plugins to upload necessary logs into oci object storage. Future actions are needed for production.
* Push or Pull the updated images from OCIR of oracle OCI. So we can keep updating new docker images.Please refer [my another note](https://www.henryxieblogs.com/2018/10/how-to-pushpull-docker-images-into.html)
* Update EFK stack to OCIR images instead of k8s.gcr.io . As fluentd works as daemonset, so every worker node will collect specific we are looking for (ie nginx access logs)
* Put fluent.conf into the configmap of K8S, so it would be safe and easy to update
