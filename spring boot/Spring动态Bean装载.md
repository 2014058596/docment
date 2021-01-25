- #### ImportSelector && DeferredImportSelector

  - 　该接口文档上说的明明白白，其主要作用是收集需要导入的配置类，如果该接口的实现类同时实现EnvironmentAware， BeanFactoryAware ，BeanClassLoaderAware或者ResourceLoaderAware，那么在调用其selectImports方法之前先调用上述接口中对应的方法，如果需要在所有的@Configuration处理完在导入时可以实现DeferredImportSelector接口。

  - 这个接口在哪里调用呢？我们可以来看一下ConfigurationClassParser这个类的processImports方法

  - ```java
    private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
                Collection<SourceClass> importCandidates, boolean checkForCircularImports) {
    
            if (importCandidates.isEmpty()) {
                return;
            }
    
            if (checkForCircularImports && isChainedImportOnStack(configClass)) {
                this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
            }
            else {
                this.importStack.push(configClass);
                try {
                    for (SourceClass candidate : importCandidates) {　　　　　　　　　　　　//对ImportSelector的处理
                        if (candidate.isAssignable(ImportSelector.class)) {
                            // Candidate class is an ImportSelector -> delegate to it to determine imports
                            Class<?> candidateClass = candidate.loadClass();
                            ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                            ParserStrategyUtils.invokeAwareMethods(
                                    selector, this.environment, this.resourceLoader, this.registry);
                            if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {　　　　　　　　　　　　　　　　//如果为延迟导入处理则加入集合当中
                                this.deferredImportSelectors.add(
                                        new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
                            }
                            else {　　　　　　　　　　　　　　　　//根据ImportSelector方法的返回值来进行递归操作
                                String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                                Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                                processImports(configClass, currentSourceClass, importSourceClasses, false);
                            }
                        }
                        else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                            // Candidate class is an ImportBeanDefinitionRegistrar ->
                            // delegate to it to register additional bean definitions
                            Class<?> candidateClass = candidate.loadClass();
                            ImportBeanDefinitionRegistrar registrar =
                                    BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                            ParserStrategyUtils.invokeAwareMethods(
                                    registrar, this.environment, this.resourceLoader, this.registry);
                            configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                        }
                        else {　　　　　　　　　　　　　　// 如果当前的类既不是ImportSelector也不是ImportBeanDefinitionRegistar就进行@Configuration的解析处理
                            // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                            // process it as an @Configuration class
                            this.importStack.registerImport(
                                    currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                            processConfigurationClass(candidate.asConfigClass(configClass));
                        }
                    }
                }
                catch (BeanDefinitionStoreException ex) {
                    throw ex;
                }
                catch (Throwable ex) {
                    throw new BeanDefinitionStoreException(
                            "Failed to process import candidates for configuration class [" +
                            configClass.getMetadata().getClassName() + "]", ex);
                }
                finally {
                    this.importStack.pop();
                }
            }
        }
    ```

    

- #### ImportBeanDefinitionRegistrar

  - ImportBeanDefinitionRegistrar类只能通过其他类@Import的方式来加载，通常是启动类或配置类。

  - 使用@Import，如果括号中的类是ImportBeanDefinitionRegistrar的实现类，则会调用接口方法，将其中要注册的类注册成bean。

  - 实现该接口的类拥有注册bean的能力。

  - ```
     @Override
     public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                              BeanDefinitionRegistry registry) {
      
              //扫描注解
              Map<String, Object> annotationAttributes = importingClassMetadata
                  .getAnnotationAttributes(ComponentScan.class.getName());
              String[] basePackages = (String[]) annotationAttributes.get("basePackages");
      
              //扫描类
              ClassPathBeanDefinitionScanner scanner =
                      new ClassPathBeanDefinitionScanner(registry, false);
              TypeFilter helloServiceFilter = new AssignableTypeFilter(HelloService.class);
              
              scanner.addIncludeFilter(helloServiceFilter);
              scanner.scan(basePackages);
          }
    ```

    

- #### **SpringFactoriesLoader(SPI)**

  - 没搞懂

- #### **配置信息带提示**

  - ##### 编写提示信息

    - ```sh
       #pom文件
       <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
       </dependency>
      #在项目中按此路径创建一个json文件 
      #resources/META-INF/spring-configuration-metadata.json
      
      
      
      {"properties": [
          {
            "name": "swagger.basePackage",
            "type": "java.lang.String",
            "description": "文档扫描包路径。"
          },
          {
            "name": "swagger.title",
            "type": "java.lang.String",
            "defaultValue": "平台系统接口详情",
            "description": "title 如: 用户模块系统接口详情。"
          },
          {
            "name": "swagger.description",
            "type": "java.lang.String",
            "defaultValue": "在线文档",
            "description": "服务文件介绍。"
          },
          {
            "name": "swagger.termsOfServiceUrl",
            "type": "java.lang.String",
            "defaultValue": "https://www.test.com/",
            "description": "服务条款网址。"
          },
          {
            "name": "swagger.version",
            "type": "java.lang.String",
            "defaultValue": "V1.0",
            "description": "版本。"
          }
      ]}
      ```

    

#### 

#### 



