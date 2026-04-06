# controller-casc-unify

For learning CloudBees Unify. 

Use this script in an Operations Center to quickly set up a CasC source. Then it should be easy to create a controller with the needed jobs.

```
import com.cloudbees.opscenter.server.casc.config.remotes.RemoteBundles
import com.cloudbees.opscenter.server.casc.config.remotes.RemoteBundleConfiguration
import com.cloudbees.opscenter.server.casc.config.remotes.SCMBundleRetriever
import jenkins.plugins.git.GitSCMSource
import jenkins.plugins.git.traits.BranchDiscoveryTrait

// Define configurations
def sourceName = "cloudbees-casc-unify"
def repoUrl = "https://github.com/kchhan/controller-casc-unify.git"
def credentialsId = "my-git-creds"

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