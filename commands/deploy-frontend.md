# To update the image version
kubectl set image deployment/front-fintracker-deployment front-fintracker=jvrochadev/front-fintracker:3.0 -n fintracker-dev
