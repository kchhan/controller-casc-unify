# controller-casc-unify

For learning CloudBees Unify. These scripts are intended to be used in a CloudBees CI demo environment that is running in Kubernetes or Kind cluster.

## Scripts and Instructions

### Setting up a controller for Continuous Integration

These scripts are intended to be ran on the Operations Center.

1. Use this script in an Operations Center to quickly set up a CasC source. Then it should be easy to create a controller with the needed jobs.

```
import com.cloudbees.opscenter.server.casc.config.remotes.RemoteBundles
import com.cloudbees.opscenter.server.casc.config.remotes.RemoteBundleConfiguration
import com.cloudbees.opscenter.server.casc.config.remotes.SCMBundleRetriever
import jenkins.plugins.git.GitSCMSource
import jenkins.plugins.git.traits.BranchDiscoveryTrait

/*****************
 * INPUTS        *
 *****************/

def sourceName = "cloudbees-casc-unify"
def repoUrl = "https://github.com/kchhan/controller-casc-unify.git"
def credentialsId = "my-git-creds"

/*****************
 * CONFIGURE CASC SOURCE *
 *****************/

// Get the current remote bundles singleton
def remoteBundlesStore = RemoteBundles.get()
def bundles = remoteBundlesStore.getBundles()

// Create the Generic Git SCMSource
def gitSource = new GitSCMSource(repoUrl)

// (optional) Configure credentials for git repo
// gitSource.setCredentialsId(credentialsId)

// Add SCM Behaviors (Traits)
// Fetch the default traits, add branch discovery, and apply them back to the source.
def traits = gitSource.getTraits()
traits.add(new BranchDiscoveryTrait())
gitSource.setTraits(traits)

// Instantiate the SCMBundleRetriever 
// (Using the single-argument constructor identified earlier)
def retriever = new SCMBundleRetriever(gitSource)

// Create the Configuration Object
def newBundleConfig = new RemoteBundleConfiguration(sourceName, retriever)

/*****************
 * SAVE *
 *****************/

// Add to the active list and persist to disk
if (bundles.any { it.name == sourceName }) {
    println "Bundle source '${sourceName}' already exists. Skipping."
} else {
    bundles.add(newBundleConfig)
    
    // Persist the change
    remoteBundlesStore.saveAndSync()
    println "Successfully added and saved Git CasC source: ${sourceName}"
}
```

2. Use this script in the Operations Center to create a managed controller that is configured to use the CasC bundle. This particular script is taken from https://github.com/cloudbees/jenkins-scripts/blob/master/createManagedMasterK8s.groovy

```
import com.cloudbees.masterprovisioning.kubernetes.KubernetesImagePullSecret
import com.cloudbees.masterprovisioning.kubernetes.KubernetesMasterProvisioning
import com.cloudbees.opscenter.server.model.ManagedMaster
import com.cloudbees.opscenter.server.properties.ConnectedMasterLicenseServerProperty
import com.cloudbees.opscenter.server.properties.ConnectedMasterOwnerProperty

/*****************
 * INPUTS        *
 *****************/

/**
 * The controller name is mandatory
 */
String controllerName = "controller-for-unify-3"
/**
 * Location of the controller
 * Leave empty to add to the top level
 * Provide full name of the folder to place controller in the folder, e.g.
 * String controllerParent = "Controllers"
 * String controllerParent = "Division/Sub-division"
 */
String controllerParent = ""

/**
 * Following attributes may be specified. The values proposed are the default from version 2.2.9 of Controller Provisioning
 *
 * Note: If not setting properties explicitly, the defaults will be used.
 */
/* Controller */
String  controllerDisplayName = ""
String  controllerDescription = ""
String  controllerPropertyOwners = ""
Integer controllerPropertyOwnersDelay = 5

/* Controller Provisioning */
Integer k8sDisk = 50
Integer k8sMemory = 3072
/**
 * Since version 2.235.4.1, we recommend not using the Heap Ratio. Instead add `-XX:MinRAMPercentage` and 
 * `-XX:MaxRAMPercentage` to the Java options. For example, a ratio of 0.5d translate to a percentage of 50: 
 * `-XX:MinRAMPercentage=50.0 -XX:MaxRAMPercentage=50.0`
 *
 * See https://support.cloudbees.com/hc/en-us/articles/204859670-Java-Heap-settings-best-practice and 
 * https://docs.cloudbees.com/docs/release-notes/latest/cloudbees-ci/modern-cloud-platforms/2.235.4.1.
 */
Double  k8sMemoryRatio = null
Double  k8sCpus = 1
String  k8sFsGroup = "1000"
Boolean k8sAllowExternalAgents = false
String  k8sClusterEndpointId = "default"
String  k8sEnvVars = ""
String  k8sJavaOptions = "-XX:MinRAMPercentage=50.0 -XX:MaxRAMPercentage=50.0"
String  k8sJenkinsOptions = ""
// String  k8sImage = 'CloudBees CI - Managed Master - 2.289.1.2'
List<KubernetesImagePullSecret> k8sImagePullSecrets = Collections.emptyList()
// Example: 
//   def k8sImagePullSecret1 = new KubernetesImagePullSecret(); k8sImagePullSecret1.setValue("useast-reg")
//   List<KubernetesImagePullSecret> k8sImagePullSecrets = Arrays.asList(k8sImagePullSecret1)
Integer k8sLivenessInitialDelaySeconds = 300
Integer k8sLivenessPeriodSeconds = 10
Integer k8sLivenessTimeoutSeconds = 10
Integer k8sReadinessInitialDelaySeconds = 30
Integer k8sReadinessFailureThreshold = 100
Integer k8sReadinessTimeoutSeconds = 5
String  k8sStorageClassName = ""
String  k8sSystemProperties = ""
String  k8sNamespace = ""
String  k8sNodeSelectors = ""
Long    k8sTerminationGracePeriodSeconds = 1200L
String  k8sYaml = ""

/**
 * cascBundle (optional). Configuration as Code Configuration Bundle
 */
String cascBundle = "main/controller-for-unify"

/*****************
 * CREATE CONTROLLER *
 *****************/
def jenkins = Jenkins.get()
ModifiableTopLevelItemGroup parent = controllerParent?.trim() ? jenkins.getItemByFullName(controllerParent) : jenkins
if (!parent) {
    println("Cannot find parent '${controllerParent}'.")
    return
}
/**
 * Create a Managed Controllers with just a name (this will automatically fill required values for id, idName, grantId, etc...)
 * Similar to creating an iten in the UI
 */
ManagedMaster newInstance = parent.createProject(jenkins.getDescriptorByType(ManagedMaster.DescriptorImpl.class), controllerName, true)
newInstance.setDescription(controllerDescription)
newInstance.setDisplayName(controllerDisplayName)

/************************
 * CONFIGURATION BUNDLE *
 ************************/
if(cascBundle?.trim()) {
    // Properties must follow this order
    // Note: ConnectedMasterTokenProperty supported since 2.289.1.2. For earlier version, comment out the following.
    newInstance.getProperties().replace(new com.cloudbees.opscenter.server.casc.config.ConnectedMasterTokenProperty(hudson.util.Secret.fromString(UUID.randomUUID().toString())))
    // Note: ConnectedMasterCascProperty supported since 2.277.4.x. For earlier version, comment out the following.
    newInstance.getProperties().replace(new com.cloudbees.opscenter.server.casc.config.ConnectedMasterCascProperty(cascBundle))
}

/********************
 * CONFIGURE CONTROLLER *
 ********************/

/**
 * Configure the Controller provisioning details. Refer to the `config.xml` for more details.
 * Similar to configuring a Managed Controller from the UI
 */
KubernetesMasterProvisioning controllerProvisioning = new KubernetesMasterProvisioning()
controllerProvisioning.setDomain(controllerName.toLowerCase())

/**
 * Apply Managed Controller provisioning configuration (similar to what is configured through the Managed Controller UI)
 * Note: If not setting properties explicitly, the defaults will be used.
 */
controllerProvisioning.setDisk(k8sDisk)
controllerProvisioning.setMemory(k8sMemory)
if(k8sMemoryRatio) {
    controllerProvisioning.setHeapRatio(new com.cloudbees.jce.masterprovisioning.Ratio(k8sMemoryRatio))
    /**
     * For versions earlier than 2.235.4.1 (Master Provisioning plugin 2.5.6), use setRatio
     * controllerProvisioning.setRatio(k8sMemoryRatio)
     */
}
controllerProvisioning.setCpus(k8sCpus)
controllerProvisioning.setFsGroup(k8sFsGroup)
controllerProvisioning.setAllowExternalAgents(k8sAllowExternalAgents)
controllerProvisioning.setClusterEndpointId(k8sClusterEndpointId)
controllerProvisioning.setEnvVars(k8sEnvVars)
controllerProvisioning.setJavaOptions(k8sJavaOptions)
controllerProvisioning.setJenkinsOptions(k8sJenkinsOptions)
// controllerProvisioning.setImage(k8sImage)
controllerProvisioning.setImagePullSecrets(k8sImagePullSecrets)
controllerProvisioning.setLivenessInitialDelaySeconds(k8sLivenessInitialDelaySeconds)
controllerProvisioning.setLivenessPeriodSeconds(k8sLivenessPeriodSeconds)
controllerProvisioning.setLivenessTimeoutSeconds(k8sLivenessTimeoutSeconds)
controllerProvisioning.setReadinessInitialDelaySeconds(k8sReadinessInitialDelaySeconds)
controllerProvisioning.setReadinessFailureThreshold(k8sReadinessFailureThreshold)
controllerProvisioning.setReadinessTimeoutSeconds(k8sReadinessTimeoutSeconds)
controllerProvisioning.setStorageClassName(k8sStorageClassName)
controllerProvisioning.setSystemProperties(k8sSystemProperties)
controllerProvisioning.setNamespace(k8sNamespace)
controllerProvisioning.setNodeSelectors(k8sNodeSelectors)
controllerProvisioning.setTerminationGracePeriodSeconds(k8sTerminationGracePeriodSeconds)
controllerProvisioning.setYaml(k8sYaml)

/**
 * Provide Controller item general configuration (similar to what is configured through the Controller UI)
 * Note: If not setting properties explicitly, the defaults will be used.
 */
if (controllerPropertyOwners != null && !controllerPropertyOwners.isEmpty()) {
    newInstance.getProperties().replace(new ConnectedMasterOwnerProperty(controllerPropertyOwners, controllerPropertyOwnersDelay))
}
newInstance.getProperties().replace(new ConnectedMasterLicenseServerProperty(new ConnectedMasterLicenseServerProperty.DescriptorImpl().defaultStrategy()))

/**
 * Save the configuration
 */
newInstance.setConfiguration(controllerProvisioning)
newInstance.save()

/**
 * Retrieve the controller from the API and print the details of the created Managed Controller
 */
def instance = jenkins.getItemByFullName(newInstance.fullName, ManagedMaster.class)
println "${instance.fullName}"
println " id: ${instance.id}"
println " idName: ${instance.idName}"

/******************************
 * PROVISION AND START CONTROLLER *
 ******************************/

instance.provisionAndStartAction()
println "Started the controller..."
return
```

At the time of writing (April 2026), CloudBees Unify does not have an API to create an Integration. The Integration Configuration requires input through the UI. You can find the Integration page in CloudBees Unify at (Configurations -> Integrations -> Create Integration).

### Setting up a CloudBees Unify Insights configuration

These scripts are intended to be ran on the Operations Center. An Operations Center Insights Integration should be created in CloudBees Unify.

1. This script is used to ensure that the cloudbees-platform-insights plugin is installed before configuring the plugin. This script will perform a safe restart after installing the plugin.

```
/*****************
 * INPUTS        *
 *****************/

def pluginName = "cloudbees-platform-insights" 

/*****************
 * INSTALL PLUGIN *
 *****************/

def instance = Jenkins.getInstance()
def pm = instance.getPluginManager()
def uc = instance.getUpdateCenter()

if (!pm.getPlugin(pluginName)) {
    println "Attempting to install ${pluginName}..."
    def plugin = uc.getPlugin(pluginName)
    if (plugin) {
        // Deploy returns a Future; .get() waits for the download to finish
        plugin.deploy().get() 
        println "Installation of ${pluginName} complete. Triggering Safe Restart..."
        
        // Triggers a safe restart
        instance.safeRestart()
    } else {
        println "Plugin ${pluginName} not found."
    }
} else {
    println "Plugin ${pluginName} is already installed."
}
```

2. Use an Access Key from CloudBees Unify with this script to set up the integration. CloudBees Unify (Configurations -> Integrations -> Create Integration)

```
import hudson.util.Secret
import com.cloudbees.jenkins.plugins.insights.configuration.context.InsightConfigurationContext
import com.cloudbees.jenkins.plugins.insights.configuration.InsightConfiguration
import com.cloudbees.jenkins.plugins.insights.configuration.oc.InsightPropagation

/*****************
 * INPUTS        *
 *****************/

// Get accessKey from CloudBees Unify (Configurations -> Integrations -> Create Integration)
def accessKey = Secret.fromString("XXXXXXXXXXXXXXXX")
def fetchHistoricalData = true
def platfromUrl = "https://api.cloudbees.io/"

// (optional) Operations Center can propagate the configuration to connected controllers
def propagate = true
def include = "*"
def exclude = ""

/*****************
 * CONFIGURE INSTANCE *
 *****************/

// Create new insights configuration
def insightsContext = new InsightConfigurationContext(accessKey, fetchHistoricalData, platfromUrl)
// Retrieve the global configuration instance for Platform Insights
def insightsConfig = Jenkins.get().getDescriptorByType(InsightConfiguration.class)

// Check if the instance is an Operations Center and if propagating is desired
if (insightsConfig.isConfigPropagationSupported() && propagate == true) {
    insightsConfig.setConfigPropagation(new InsightPropagation(include, exclude))
}

/*****************
 * SAVE CONFIGURATION *
 *****************/

insightsConfig.fromOC(insightsContext)
insightsConfig.save()
println "Successfully configured an Insights configuration."
```