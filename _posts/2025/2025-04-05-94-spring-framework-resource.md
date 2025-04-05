---
layout: post
title:  "94. Spring Framework: Resource"
date:   2025-04-05 08:02:00 +0000
category: technical
---
- [1. Java Class loader \[1\]](#1-java-class-loader-1)
  - [1.1. JAR hell](#11-jar-hell)
  - [1.2. CLASSPATH env variable](#12-classpath-env-variable)
  - [1.3. Using ClassLoader to load resource](#13-using-classloader-to-load-resource)
- [2. Maven project structure](#2-maven-project-structure)
- [3. Spring Framework: Resource \[2\]](#3-spring-framework-resource-2)
  - [3.1. Using prefix](#31-using-prefix)
- [4. References](#4-references)

This article talks about how to read resources (file) in Spring Boot applicaiton. Giving 2 ways to read resource and the different between them: Using ClassLoader (core Java) and using ResourceLoader (core Spring)

Note: using ChatGPT + copy some contents from reference source. 

### 1. Java Class loader [1]
The Java class loader, part of the Java Runtime Environment, dynamically loads Java classes into the Java Virtual Machine. Usually classes are only loaded on demand. The virtual machine will only load the class files required for executing the program.

This loading is typically done "on demand", in that it does not occur until the class is called by the program. A class with a given name can only be loaded once by a given class loader.

When the JVM is started, three class loaders are used:

- Bootstrap class loader: The bootstrap class loader loads the core Java libraries located in the <JAVA_HOME>/jre/lib (or <JAVA_HOME>/jmods> for Java 9 and above) directory. This class loader, which is part of the core JVM, is written in native code. The bootstrap class loader is not associated with any ClassLoader object, for instance, StringBuilder.class.getClassLoader() returns null.
- Extensions class loader: loads the code in the extensions directories (<JAVA_HOME>/jre/lib/ext,[5] or any other directory specified by the java.ext.dirs system property).
- System class loader: The system class loader loads code found on **java.class.path** env variable, which maps to the **CLASSPATH** environment variable.

#### 1.1. JAR hell
JAR hell used to describe all the various ways in which the classloading process can end up not working. JAR hell can occur are:

- Accidental presence of two different versions of a library installed on a system. This will not be considered an error by the system. Rather, **the system will load classes from one or the other library**. Adding the new library to the list of available libraries instead of replacing it may result in the application still behaving as though the old library is in use, which it may well be.
- Multiple libraries or applications require different versions of library **foo**. If versions of library **foo** use the same class names, there is no way to load the versions of library **foo** with the same class loader.

#### 1.2. CLASSPATH env variable
Classpath is a parameter in the Java Virtual Machine or the Java compiler that **specifies the location of user-defined classes, packages, and resources (if using Maven project style)**. The parameter may be set either on the command-line, or through an environment variable.

Setting the Classpath via the Command Line
```bash
-- Window based machine
java -classpath /path/to/class/files MyProgram

-- Linux
java -cp /path/to/class/files MyProgram

```

#### 1.3. Using ClassLoader to load resource 
```java 
public List<String> readFileWithClassLoader(String filePath) throws IOException {
        List<String> res = new ArrayList<>();
        // return ClassLoader that use by resourceLoeader. It's jdk.internal.loader.ClassLoaders$AppClassLoader
        ClassLoader classLoader = resourceLoader.getClassLoader();
        if(classLoader == null){
            System.out.println("classloader is not exist");
            throw new NullPointerException("classloader is not exist");
        }
        System.out.println("classLoader: " + classLoader.getClass().getName());
        try (BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(classLoader.getResourceAsStream(filePath)))){
            String line;
            while ((line = bufferedReader.readLine()) != null) {
                res.add(line);
            }
        }
        return res;
    }
```

### 2. Maven project structure 

Classpath Locations:

**Source Code:**

- src/main/java: Contains the main application's source code. All .class files generated during compilation are included in the classpath.
- src/test/java: Includes test classes, which are added to the classpath during testing.

**Resources:**
- src/main/resources: Non-code resources for the application (e.g., application.properties, XML files) are included in the classpath automatically.
- src/test/resources: Resources specifically for testing are added to the classpath when running tests.

**Compiled Code:**
- target/classes: After compiling your application, the .class files are stored here. Maven ensures this directory is part of the classpath. 
- target/test-classes: After compiling test code, the generated .class files are stored here, included during test execution.

**Dependencies:**
Maven automatically downloads and manages dependencies specified in the pom.xml file. These JAR files are stored in the local repository (~/.m2/repository) and included in the classpath during builds.

![Maven_project_structure_output](/assets/images/2025/95_maven_project_structure_output.png)

As above image, our classpath is **target/classes**, and folder input inside **src/main/resources** folder is placed under **target/classes**

### 3. Spring Framework: Resource [2]
For more detail on Spring Framework Resource, read at [2].

The *ResourceLoader* interface is meant to be implemented by objects that can return (that is, load) *Resource* instances.

All application contexts implement the ResourceLoader interface. Therefore, *all application contexts may be used to obtain Resource instances*.

When you call getResource() on a specific application context, and the location path specified doesn’t have a specific prefix, you get back a Resource type that is appropriate to that particular application context. For example, *FileSystemXmlApplicationContext* instance, it would return a FileSystemResource. For a *WebApplicationContext*, it would return a *ServletContextResource* (this is the case for web development, using *spring-boot-starter-web*). It would similarly return appropriate objects for each context.

As a result, you can load resources in a fashion appropriate to the particular application context.

#### 3.1. Using prefix
On the other hand, you may also force *ClassPathResource* to be used, regardless of the application context type, by specifying the special **classpath:** prefix. Similarly, you can force a UrlResource to be used by specifying any of the standard java.net.URL prefixes like file and https

![Resource_prefix](/assets/images/2025/95_resource_prefix.png)

When you don't specify any prefix, Resource implementation class will depend on ApplicationContext.

Personally, I prefer this approach, as it gives us more information explicitly.


### 4. References 
1. [Java Class Loader](https://en.wikipedia.org/wiki/Java_class_loader)
2. [Spring Framework: Resource](https://docs.spring.io/spring-framework/reference/core/resources.html)



