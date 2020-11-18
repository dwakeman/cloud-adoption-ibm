## Using your custom domain with OpenShift

Now that you have your spiffy new custom domain, let's put it to work!  The steps documented here assume that you have done the following:

1. Registered your custom domain
1. Provisioned an instance of IBM Cloud Internet Services and configured it to manage your domain
1. Ordered origin certificates in IBM Cloud Internet Services
1. Ordered edge certificates in IBM Cloud Internet Services


I used this page in the OpenShift docs to create the secured route for my application.

[Creating an edge route with a custom certificate](https://docs.openshift.com/container-platform/4.1/networking/routes/secured-routes.html#nw-ingress-creating-an-edge-route-with-a-custom-certificate_secured-routes)