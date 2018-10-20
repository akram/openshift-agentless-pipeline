# openshift-agentless-pipeline

An agentless pipeline build relying on an S2I image.
The build runs in jenkins (agent none) and creates if required the sub-build using s2i.
Then the build is monitored and when finished, it tags and deploy the app.
