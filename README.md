# controller-casc-unify

For learning CloudBees Unify. 

Use this script in an Operations Center to quickly set up a CasC source. Then it should be easy to create a controller with the needed jobs.

```
import com.cloudbees.opscenter.server.casc.config.remotes.RemoteBundles
import com.cloudbees.opscenter.server.casc.config.remotes.RemoteBundleConfiguration
import com.cloudbees.opscenter.server.casc.config.remotes.SCMBundleRetriever
import jenkins.plugins.git.GitSCMSource

// Get the current remote bundles singleton
def remoteBundlesStore = RemoteBundles.get()
def bundles = remoteBundlesStore.getBundles()

// 1. Define a generic Git SCMSource
def repoUrl = "https://github.com/kchhan/controller-casc-unify.git"

// (optional) Set credentials if needed
// def credentialsId = "my-git-creds"

def gitSource = new GitSCMSource(repoUrl)
// gitSource.setCredentialsId(credentialsId)

// 2. Instantiate the SCMBundleRetriever using the single-argument constructor
def retriever = new SCMBundleRetriever(gitSource)

// Note: If your CasC bundle is NOT in the root of the repository, you will likely 
// need to set the path using a setter method on the retriever.
// E.g., retriever.setBundlePath("bundles/production") or retriever.setPath("...")
// (You can use retriever.properties.keySet() to find the exact property name if needed)

// 3. Create the Configuration Object
def sourceName = "cloudbees-casc-unify"
def newBundleConfig = new RemoteBundleConfiguration(sourceName, retriever)

// 4. Add to the active list and persist to disk
if (bundles.any { it.name == sourceName }) {
    println "Bundle source '${sourceName}' already exists. Skipping."
} else {
    bundles.add(newBundleConfig)
    
    // Crucial: Save the configuration so it persists across reboots
    remoteBundlesStore.save()
    println "Successfully added and saved Git CasC source: ${sourceName}"
}
```