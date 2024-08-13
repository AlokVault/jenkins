# jenkins

# Involves

1.  It reads the commit user and sends and email if the build fails
2.  It reads the repository and do the maven build 
3.  Once build is done, it archive the image version in a text file to be read in another jenkins job
4.  It has Post step also, to run if any build fails
5.  Then it publishes the images to a azure container registeries to be pulled from anywhere
6.  Then it tags the version in gitlab to maintian the version history
7.  Once the jenkins file is done with a success, it triggers the google chat notification to notify the team