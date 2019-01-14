
# Writing kubectl plugins

* TODOs:
  * Migrate to Github Actions
  * Migrate to krew
  * Write blog
  * Feedback
  * Publish:
    * Medium    

# Blog structure

* Topic: 
  * Writing kubectl plugins in Go
* Goals: 
  * Explain how kubectl plugins work & how they can be implemented
  * Also showcase Github Actions & [krew](https://github.com/GoogleContainerTools/krew) in the process
* Audience:
  * Kubernetes users interested in using kubectl plugins
  * Kubernetes users interested in writing kubectl plugins
* Structure:
  * Introduction
    * Explain context: kubectl & kubectl plugins 
    * Show what kubectl plugins can do based on existing examples
    * Links to related blog posts
    * Outline structure of this blog post 
  * Middle: 
    * Overview 
      * Picture how kubectl plugins are integrated in kubectl
      * Explanation of the kubectl plugin interface
        * Env vars / plugin in path...
     * Walkthrough concrete code example based on kubectl os
     * Building kubectl plugins with Github Actions & publishing to krew
     * Installing & using the kubectl os plugin with krew
 * End: 
   * Conclusion
