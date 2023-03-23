## Create a Cloud Source Repository using Cloud Shell
```
gcloud source repos create GCPKenobi-repo
```

## Clone newly created repo into Cloud Shell
```
gcloud source repos clone GCPKenobi-repo
```

## Push from Cloud Shell to Cloud Source Repository
CD into the repository
```
cd GCPKenobi-repo
```

Create a new file to push to the repo
```
echo 'Hello there! General Kenobi!' > generalkenobi.txt
```

Update the config with your credentials
```
git config --global user.email GCPKenobi@gmail.com
git config --global user.name GCPKenobi
```

Stage the newly created file
```
git add generalkenobi.txt
```

Commit the file with a message
```
git commit -m 'First commit of generalkenobi.txt'
```

Push the file to the Cloud Source Repository
```
git push origin master
```

## Browse Files in Cloud Source Repository
### From Cloud Shell
Use the below command in Cloud Shell to get a url to the Cloud Source Repository.
```
gcloud source repos list
```

