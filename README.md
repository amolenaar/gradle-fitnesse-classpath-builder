# Simple class path generation for FitNesse

FitNesse is a well known tool for software acceptance testing. One of the harder things to do is execute your application from FitNesse. For this, at least in the case of Java/JVM applications, you'll need to tell FitNesse where the libraries and folders are that make up your application class path.

Since we're used to tools like Gradle, that do all the dependency resolution just fine, it's a hassle to keep dependencies up to date manually.

The FitNesse wiki uses a simple file structure to store its page content. The directory name resembles the page name, the wiki markup is stored in a file named `content.txt` and metadata is stored in a file `properties.xml`.

The approach we take is that we create a new page with class path directives. There's no need to consider the properties file, since it has some sensible defaults. All the page will contain lines formatted like `!path some/archive.jar`.

In Groovy code this looks like:

```
class WriteFitNesseClasspath extends DefaultTask {
  @Input
  def classpath

  @OutputDirectory
  File pagePath

  @TaskAction
  def generatePage() {
    def contentTxt = project.file("${pagePath}/content.txt")
    contentTxt.createNewFile()
    contentTxt.withWriter { writer ->
      writer.writeLine("!path ${project.sourceSets.main.output.classesDir}")
      writer.writeLine("!path src/main/resources")
      classpath.each { d ->
        writer.writeLine("!path $d")
      }
    }
  }
}
```

Of course this needs to be wired with a task that can be executed as part of the build cycle.

```
task("writeFitNesseClasspath", type: WriteFitNesseClasspath) {
  classpath = project.configurations.fitnesse + configurations.runtime

  pagePath = project.file("FitNesseRoot/GradleClasspath")
}

project.tasks.getByName("clean").dependsOn("cleanWriteFitNesseClasspath")

task("wiki", type: JavaExec) {
  dependsOn writeFitNesseClasspath
  dependsOn compileJava
  classpath configurations.fitnesse
  main "fitnesseMain.FitNesseMain"
  args "-p", "8000", "-e", "0"
}


```

At this point we have:

1. A Gradle configuration that generates a FitNesse page
2. A FitNesse page that does not do a thing...

The final step is to include the GradleClasspath page in the (toplevel) suite containing your acceptance tests.

A simple `!include .GradleClasspath` should do the trick.
