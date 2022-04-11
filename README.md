##CI/CD pipelines for ASP.NET 4.8 with AWS CodePipeline and AWS EC2 Instance 

 

As customers migrate ASP.NET (on .NET Framework) applications to AWS, many choose to deploy these apps with AWS EC2 (Windows Server), which provides a managed .NET platform to deploy, scale, and update the apps. Customers often ask how to create CI/CD pipelines for these ASP.NET 4.8 (.NET Framework) apps without needing to set up or manage Jenkins instances or other infrastructure. 

You can easily create these pipelines using AWS CodePipeline as the orchestrator, AWS CodeBuild for performing builds, AWS CodeDeploy for automating software deployments to a variety of compute services like Amazon EC2 and BitBucket or other systems for source control. This blog post demonstrates how to set up a simplified CI/CD pipeline that you could expand on later to include unit tests, using a BitBucket repository for source control. 

 
Creating a project and adding a buildspec.yml file 

The first step in setting up this simplified CI/CD pipeline is to create a project and add a buildspec.yml file. 

Creating or choosing an ASP.NET web application (.NET Framework) 

First, either create a new ASP.NET Web Application (.NET Framework) project or choose an existing application to use. You can choose MVC, Web API, or even Web Forms project types based on ASP.NET 4.8 Whichever type you choose, make sure it builds and runs locally. 

To set up your first CodePipeline for an ASP.NET (.NET Framework) application, you may wish to use a simple app that doesn’t require databases or other resources and which consists of a single project. The following screenshot shows the project type to choose when you create a new project in Visual Studio 2019. 

Visual Studio 2019's Create New Project dialog window showing "ASP.NET Web Application (.NET Framework)" project type selected. 

Visual Studio Create New Project dialog 

 

Team Explorer – Repository Settings 

After adding a .gitignore file and optionally connecting Visual Studio to BitBucket, push your code up to the remote in BitBucket using either git push or Team Explorer. After pushing your changes, you can use the BitBucket management console in your browser to verify that all your files are there. 

Adding a buildspec.yml file to your project 

CodeBuild, which does the actual compilation, essentially launches a container using a docker image you specify, then runs a series of commands to install any required software and perform the actual build or tests that you want. Finally, it takes whatever output files you specify—artifacts—and uploads them in a .zip file to Amazon S3 for the next stage of the CodePipeline pipeline. The commands that CodeBuild executes in the container are specified in a buildspec.yml file, which is part of the source code of your project. You can also add it directly to the CodeBuild configuration, but it’s more convenient to edit and track in source control. When running CodeBuild with Windows containers, the default shell for these commands is PowerShell. 

Add a plain text file to the root of your ASP.NET project named buildspec.yml and then open the file in an editor. Ensure you add the file to your project to easily find and edit it later. For details on the structure and contents of buildspec.yml files, refer to the CodeBuild documentation. 

You can use the following buildspec.yml file and simply replace the values for PROJECT and DOTNET_FRAMEWORK with the name and .NET Framework target version for your project. 

version: 0.2 

  

env: 

  variables: 

    PROJECT: test 

    DOTNET_FRAMEWORK: 4.8 

phases: 

  build: 

    commands: 

      - nuget restore 

      - msbuild $env:PROJECT.csproj /p:TargetFrameworkVersion=v$env.DOTNET_ FRAMEWORK /p:Configuration=QARelease /p:DeployIisAppPath="app-qa.sales.garden" /p:PackageAsSingleFile=false  

artifacts: 

  files: 

    - '**/*' 

YAML 

Walkthrough of the buildspec commands 

Looking at the buildspec.yml file above, you can see that the only phase defined for this sample application is build. If you need to perform some action either before or after the build, you can add pre_build and post_build phases. 

The first command executed in the build phase is nuget restore to download any NuGet packages your project references. Then, MS build kicks off the build itself. Using the /t:Package parameter generates the web deployment folder structure that Elastic Beanstalk expects for ASP.NET Framework applications, and includes the archive.xml, parameters.xml, and systemInfo.xml files. 

By default, the output of this type of build is a .zip file. However, when used in conjunction with CodePipeline, CodeBuild always zips up the artifact files that you specify, even if they’re already zipped. To avoid this double zipping, use the /p:PackageAsSingleFile=false parameter, which outputs the folder structure in a folder called Archive instead. The /p:OutDir parameter specifies where MSBuild should write the files. This example uses C:\codebuild\artifacts\. 

Finally, in the artifacts node, specify which files (or artifacts) CodeBuild should compress and provide to CodePipeline. The sample above includes all the files (the ‘**/*’). 

After you finish editing the buildspec.yml file, commit and push your changes to ensure the file is in your BitBucket repository. 

Adding two more files you have add (appspec.yml and StageArtifact.ps1) 

appspec.yml must be placed in the root of the directory structure of an application's source code. 

You can use the following appspec.yml file 

version: 0.0 

os: windows 

files: 

 source: \ 

                  destination: C:\Users\Administrator\jenkins 

hooks: 

    BeforeInstall: 

 location: .\StageArtifact.ps1 

You can use the following StageArtifact.ps1 file: 

if ($pshome -like "*syswow64*") { 

Write-Warning "Restarting script under 64 bit powershell" 

& (join-path ($pshome -replace "syswow64", "sysnative") powershell.exe) -file ` 

(join-path $psscriptroot $myinvocation.mycommand) @args 

exit $lastexitcode 

} 
$webAppName = "MyWebApp" 

$website = "test" 

$webAppPath = "C:\Users\Administrator\jenkins\" 

 Stop IIS Server  

Stop-IISSite -Name "Default Website"  

if (-not (Test-Path $webAppPath)) 

{ 

New-Item -ItemType directory -Path $webAppPath 

}else{ 

  Get-ChildItem $webAppPath -Recurse | Remove-Item 

} 

Copy-Item -Path "C:\Users\Administrator\jenkins\*" -Destination "C:\inetpub\wwwroot\test " -Recurse 

 Start IIS Server 

Start-IISSite -Name "Default Website" 

 

Configure Windows Server for deploying artifacts of Codebuild. 

Attach IAM Role to Windows Server (salesgardenqaweb) have the following IAM policies. (AmazonEC2RoleforAWSCodeDeploy and AmazonS3FullAccess) 

Install CodeDeploy agent on the Server. You can refer for the details.  

Creating the CI/CD pipeline 

Next, create the CodePipeline pipeline. 

Adding the source stage 

Now that your source code is in BitBucket Repository, create your pipeline: 

In your browser, navigate to the CodePipeline management console. 

Choose Create pipeline and give your pipeline a name. To keep things simple, you might want to use the same name as your BitBucket repo. 

Choose Next. 

Under Source, choose BitBucket. 

Select your repository name from the drop-down, and choose the branch you wish to use. If you haven’t added any branches, your only choice will be the master branch. 

Creating the build stage 

Next, create the build stage: 

After choosing Next, select AWS CodeBuild as the build provider. 

Select your region, then choose Create project, which will open CodeBuild in another browser window. 

In the CodeBuild window, you can optionally assign your build project a name and description. 

Under Environment, select the Custom image option, and select Windows as the environment type. 

For building ASP.NET 4.8 (.NET Framework) web projects, it’s easiest to start out with Microsoft’s .NET Framework SDK docker image, which they host on their registry. 
Select Other registry, and use mcr.microsoft.com/dotnet/framework/sdk:4.8 

For details about the .NET Framework SDK container image, see the container image page on Dockerhub. The SDK includes the Visual Studio Build Tools, the NuGet CLI, and ASP.NET Web Targets. 

Next, choose a group name for Amazon CloudWatch logs under Logs (near the bottom of the page). This will output detailed build logs for each build to CloudWatch. Leave the rest of the settings as they are. 

Then choose Continue to CodePipeline to save the CodeBuild configuration and return to the CodePipeline wizard’s Add build stage step. Ensure your newly created build project is specified in Project name, then choose Next. 

Adding the deploy stage 

In the Add deploy stage step: 

Select AWS CodeDeploy as the Deploy provider. 

Select your Application name. 

In the Application name field, select the sg-splash-qa-app application you previously deployed. 

Select the Deployment Group (sg-splash-qa-dg) you previously deployed and choose Next. 

Review all your settings and choose Create pipeline. 

Testing out the pipeline 

To test out your pipeline, make an easily visible change to your application’s code, such as adding some text to the home page. Then, commit your changes and push. 

Within a few moments, the Source stage in your pipeline should move to in progress, followed by the Build stage. It can take 10 minutes or more for the build stage to complete, and then the Deploy stage should finish quickly. 

 

Screenshot of successful CodePipeline pipeline 

Conclusion 

In this blog post, I showed you how to create a simple CI/CD pipeline for ASP.NET 4.8 web applications, built with the .NET Framework, using AWS services including BitBucket, CodePipeline, CodeBuild and CodeDeploy. You can extend this pipeline with additional build actions for things like unit tests, or by adding manual approval steps. 

 
