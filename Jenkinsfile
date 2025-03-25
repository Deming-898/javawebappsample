import groovy.json.JsonSlurper

// 解析 FTP 发布信息
def getFtpPublishProfile(def publishProfilesJson) {
  def pubProfiles = new JsonSlurper().parseText(publishProfilesJson)
  for (p in pubProfiles) {
    if (p['publishMethod'] == 'FTP') {
      return [url: p.publishUrl, username: p.userName, password: p.userPWD]
    }
  }
  return null
}

node {
  withEnv(['AZURE_SUBSCRIPTION_ID=5a8cf25d-3cb1-4df5-be6a-d01a4e064195',
        'AZURE_TENANT_ID=cf078ecd-4131-4541-a92a-0c7ee308aeb7']) {

    stage('init') {
      checkout scm
    }
  
    stage('build') {
      sh 'mvn clean package'
    }
  
    stage('deploy') {
      def resourceGroup = 'jenkins-get-started-rg'
      def webAppName = 'jenkins-webapp-8848'

      // 登录 Azure
      withCredentials([usernamePassword(credentialsId: 'AzureServicePrincipal', passwordVariable: 'AZURE_CLIENT_SECRET', usernameVariable: 'AZURE_CLIENT_ID')]) {
        sh '''
          az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET -t $AZURE_TENANT_ID
          az account set -s $AZURE_SUBSCRIPTION_ID
        '''
      }

      // 获取 Web App 发布信息
      def pubProfilesJson = sh(script: "az webapp deployment list-publishing-profiles -g ${resourceGroup} -n ${webAppName}", returnStdout: true).trim()
      def ftpProfile = getFtpPublishProfile(pubProfilesJson)

      if (!ftpProfile) {
        error("FTP Profile not found. Deployment failed.")
      }

      // 确保 FTP URL 末尾带 `/webapps/ROOT.war`
      def ftpUploadUrl = "${ftpProfile.url}/webapps/ROOT.war"

      // 上传 WAR 包
      sh "curl -T target/calculator-1.0.war ${ftpUploadUrl} -u '${ftpProfile.username}:${ftpProfile.password}'"

      // 退出 Azure
      sh 'az logout'
    }
  }
}
