# openshift-agentless-pipeline

An agentless pipeline build relying on an S2I image.
The build runs in jenkins (agent none) and creates if required the sub-build using s2i.
Then the build is monitored and when finished, it tags and deploy the app.



To allo the Jenkins pipeline to create the projects for staging, uat and prod, we need to:
```
oc adm policy add-cluster-role-to-user self-provisioner  system:serviceaccount:php-pipeline:jenkins
```



```
# oc create secret generic git-repo-secret --from-literal=username=user \
#                                    --from-literal=password=password

oc create secret generic git-repo-secret --from-file=ssh-privatekey=~/.ssh/id_rsa
GIT_REPO=https://github.com/akram/openshift-agentless-pipeline.git
oc new-build --name=my-pipeline --strategy=pipeline \
             --code=$GIT_REPO
oc env bc my-pipeline GIT_SSL_NO_VERIFY=true

oc create configmap jenkins-approval-scripts --from-file=scriptApproval.xml
oc set volume --add dc/jenkins -t configmap --mount-path=/var/lib/jenkins/scriptApproval.xml \
              --sub-path=scriptApproval.xml --configmap-name=jenkins-approval-scripts \
              --name=script-approval

```


