# Deploying and running Karate test after en QUA for any project (TNR)

This is how we can test in the TNR passes after we do a new merge into the main branch of any project 

### 1. We merge the merge request with the main
### 2. We wait till the pipeline pass
### 3. At this level, we have successfully deployed in DEV, and automatically auto synchonized in argoCD 
### 4. We click on changelog generation, which after will generate the new tag automatically 
### 5. At this level, a new pipeline will be created by HARVEST_CI_TOKEN
### 6. The piepline will run and will deploye in QUA, this is the max automatically deployement level
### 7. Here we can go to the karate project of our project, check into pipelines
### 8. We click on the pipelines, go to the latest morning scheduled pipelines (cron) 
### 9. We re-launch the RCI job, and if everything is good, that means that our TNR has all passed successfully
### 10. At this level, we go back to our projects pipeline, and we click manually on the run button to run the UAT, STG then PROD deployements 
### 11. Then all we need to do now is to go to argoCD, and check if the tag is good and pointing to the latest version, then we click on sync -> synchonize

And this is the perfect flow for adding, testing & deploying a new project version 
