# Overview

Here is a short example of how to test an operator bundle with the OLM.
It uses a fake operator which is really just a docker.io/busybox instance
looping and echoing, but it's enough to illustrate the steps.

# Helpful links

This guide gives you a quick howto, but you can delve deeper here

* https://github.com/operator-framework/operator-lifecycle-manager
* https://github.com/operator-framework/operator-courier
* https://github.com/operator-framework/operator-registry

# Adding a custom operator using foo-operator as an example

1. Create an account on quay.io if you do not already have one

2. Install operator-courier using pip (Python 3 required)

   ```bash
   $ pip3 install operator-courier
   ```

3. Get the get-quay-token script (it's convenient)

   ```bash
   $ wget https://raw.githubusercontent.com/operator-framework/operator-courier/master/scripts/get-quay-token
   $ chmod +x get-quay-token
   ```

4. Generate your token for quay and save it in an environment variable (you could get fancy here and pipe the output to some bash command that parses the token out without ever displaying it, but we'll keep it simple for the guide)

   ```
   $ ./get-quay-token 
   Username: tmckayus
   Password: 
   {"token":"basic blahblahnotreallymytokendG1ja="} 
  
   $ QUAY_TOKEN="basic blahblahnotreallymytokendG1ja="
   ```

5. Create a bundle from the foo-operator directory and push it to quay.io.

   The arguments to push are SOURCE_DIRECTORY QUAY_USERNAME REPOSITORY_NAME BUNDLE_VERSION QUAY_TOKEN.

   Note for now that the bundle version is arbitrary.

   ```bash
   $ operator-courier push foo-operator tmckayus foo-operator 0.1.3 "$QUAY_TOKEN"
   ```

6. When you first create a new repository in quay, it will be private. Log in to quay and make it public. You can do this from "Settings" for the repository.

7. The _os.yaml_ file in this repo contains an OperatorSource manifest, edit it to replace the registryNamespace value with your quay.io username

   ```bash
   apiVersion: operators.coreos.com/v1
   kind: OperatorSource
   metadata:
       name: my-operators
       namespace: openshift-marketplace
   spec:
       type: appregistry
       endpoint: https://quay.io/cnr
       registryNamespace: "your quay username here"
   ```

8. Create the OperatorSource object in OpenShift

   ```bash
   $ oc create -f os.yaml
   ```

9. You can verify that the operator source has been created with a corresponding deployment

   ```bash
   $ oc get OperatorSource -n openshift-marketplace
   ...
   my-operators          appregistry   https://quay.io/cnr   tmckayus   Succeeded   The object has been successfully reconciled   2m

   $ oc log -f deployment/my-operators -n openshift-marketplace
   time="2019-08-02T20:12:34Z" level=info msg="Using in-cluster kube client config" port=50051 type=appregistry
   time="2019-08-02T20:12:34Z" level=info msg="operator source(s) specified are - [https://quay.io/cnr%7Ctmckayus]" port=50051 type=appregistry
   time="2019-08-02T20:12:34Z" level=info msg="package(s) specified are - foo-operator," port=50051 type=appregistry
   ...
   ```

10. Go to the OperatorHub UI in OpenShift and you should see the foo-operator listed there under the "custom" category

# How to iterate quickly and update the foo-operator

1. Make some modification to the cluster service version file in the foo-operator directory

2. Push the new bundle version to quay.io but increment the BUNDLE_VERSION value

   ```
   $ operator-courier push foo-operator tmckayus foo-operator 0.1.4 "$QUAY_TOKEN"
   ```
3. Uninstall the current version of the operator if it's currently installed

4. Cause the pod associated with the OperatorSource to be recreated

   ```bash
   $ oc scale deployment my-operators --replicas=0 -n openshift-marketplace
   $ oc scale deployment my-operators --replicas=1 -n openshift-marketplace
   ``` 

5. You can convince yourself that the new version was picked up by looking at the log for the deployment and searching for the updated BUNDLE_VERSION

   ```bash
   $ oc log -f deployment/my-operators -n openshift-marketplace | grep 0.1.4
   ```

6. The new version of the operator bundle should now be visible in OperatorHub, go ahead and reinstall the operator
