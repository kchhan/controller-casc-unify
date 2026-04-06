# controller-casc-unify

For learning CloudBees Unify. 

Use this script in an Operations Center to quickly set up a CasC source. Then it should be easy to create a controller with the needed jobs.

```
import com.cloudbees.opscenter.server.casc.config.remotes.RemoteBundles
import com.cloudbees.opscenter.server.casc.config.remotes.RemoteBundleConfiguration
import com.cloudbees.opscenter.server.casc.config.remotes.SCMBundleRetriever
import jenkins.plugins.git.GitSCMSource
import jenkins.plugins.git.traits.BranchDiscoveryTrait

// Get the current remote bundles singleton
def remoteBundlesStore = RemoteBundles.get()
def bundles = remoteBundlesStore.getBundles()

// 1. Define your Git details
def repoUrl = "https://github.com/kchhan/controller-casc-unify.git"

// (optional) Configure a credential to be used
// def credentialsId = "my-git-creds"

// 2. Create the Generic Git SCMSource
def gitSource = new GitSCMSource(repoUrl)

// gitSource.setCredentialsId(credentialsId)

// 3. Add SCM Behaviors (Traits)
// Fetch the default traits, add branch discovery, and apply them back to the source.
def traits = gitSource.getTraits()
traits.add(new BranchDiscoveryTrait())
gitSource.setTraits(traits)

// 4. Instantiate the SCMBundleRetriever 
// (Using the single-argument constructor identified earlier)
def retriever = new SCMBundleRetriever(gitSource)

// 5. Create the Configuration Object
def sourceName = "cloudbees-casc-unify"
def newBundleConfig = new RemoteBundleConfiguration(sourceName, retriever)

// 6. Add to the active list and persist to disk
if (bundles.any { it.name == sourceName }) {
    println "Bundle source '${sourceName}' already exists. Skipping."
} else {
    bundles.add(newBundleConfig)
    
    // Persist the change
    remoteBundlesStore.saveAndSync()
    println "Successfully added and saved Git CasC source with branch discovery: ${sourceName}"
}
```