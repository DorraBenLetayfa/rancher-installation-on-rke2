INTERNAL-USE-ONLY-980a-6158
you need to go to the suse customer center and then proc=xies and get the creds from there. 
follow same steps as in documentation but create also a docker registry secret and store the creds there as well
then change the pull secret in the yaml config of the chart when you install it whith the URL of course. 

make sure that relativeURLs is set to true. 

to mirror images get the images from https://prime.ribs.rancher.io/ then go the rke2 and pick the one you need take the .txt file and create a 
script that will copy the image to harbor using scopeo.

