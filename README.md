# Gradle based classpath generation for FitNesse

FitNesse is a well known tool for software acceptance testing. If you're using Gradle as your build tool of choice, it would be great to use Gradle to provide FitNesse a classpath for executing the acceptance tests. One way of doing so is to start FitNesse (preferable with a Gradle task) and use the [FitNesse Gradle classpath](http://github.com/kukido/fitnesse-gradle-classpath) plugin.

In this blog I'll show another way to achieve the same result by generating a wiki page, which can be included in your test suite.

The FitNesse wiki uses a simple file structure to store its page content. The directory name resembles the page name, the wiki markup is stored in a file named `content.txt` and metadata is stored in a file `properties.xml`. The approach we take is that we create a new page with class path directives. There's no need to consider the properties file, since it has some sensible defaults. All the page will contain lines formatted like `!path some/archive.jar`.

Let's start with the task:

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

The final step is to include the GradleClasspath page in the (toplevel) suite containing your acceptance tests. A simple `!include .GradleClasspath` should do the trick.

A similar task to the wiki task can be created to execute the FitNesse suite as part of the build process: just change the `args`.

A working example can be found at https://github.com/amolenaar/gradle-fitnesse-classpath-builder.

