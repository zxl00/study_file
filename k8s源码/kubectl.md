## kubectl create

#### 架构图

![image-20230529094856524](https://gitee.com/root_007/md_file_image/raw/master/202305290948582.png)

​				其还有更多子命令

​				![image-20230529094934603](https://gitee.com/root_007/md_file_image/raw/master/202305290949647.png)

- create流程

  > newCmdCreate调用cobra的Run函数
  >
  > 调用RunCreate构建resourceBuilder对象
  >
  > 调用visit方法创建资源
  >
  > 底层使用resetclient和k8s-api通信

#### 源码

newCmdCreate调用cobra的Run函数

`vendor\k8s.io\kubectl\pkg\cmd\cmd.go`

```go
		{
			Message: "Basic Commands (Beginner):",
			Commands: []*cobra.Command{
				create.NewCmdCreate(f, ioStreams),
				expose.NewCmdExposeService(f, ioStreams),
				run.NewCmdRun(f, ioStreams),
				set.NewCmdSet(f, ioStreams),
			},
		},
```

> 直接由`cmd-->kubectl-->kubectl.go`点进去即可，然后看一下其中的`create.NewCmdCreate(f, ioStreams),`

```go
func NewCmdCreate(f cmdutil.Factory, ioStreams genericclioptions.IOStreams) *cobra.Command {
	o := NewCreateOptions(ioStreams)

	cmd := &cobra.Command{
		Use:                   "create -f FILENAME",
		DisableFlagsInUseLine: true,
		Short:                 i18n.T("Create a resource from a file or from stdin."),
		Long:                  createLong,
		Example:               createExample,
		Run: func(cmd *cobra.Command, args []string) {
			// 文件校验
			if cmdutil.IsFilenameSliceEmpty(o.FilenameOptions.Filenames, o.FilenameOptions.Kustomize) {
				ioStreams.ErrOut.Write([]byte("Error: must specify one of -f and -k\n\n"))
				defaultRunFunc := cmdutil.DefaultSubCommandRun(ioStreams.ErrOut)
				defaultRunFunc(cmd, args)
				return
			}
			// 完善并填充所需字段
			cmdutil.CheckErr(o.Complete(f, cmd))
			// 参数校验
			cmdutil.CheckErr(o.ValidateArgs(cmd, args))
			cmdutil.CheckErr(o.RunCreate(f, cmd))
		},
	}
    ....
```

调用RunCreate构建resourceBuilder对象

```go
cmdutil.CheckErr(o.RunCreate(f, cmd))
```

> 这个就是上边代码中最后一行代码

```go
func (o *CreateOptions) RunCreate(f cmdutil.Factory, cmd *cobra.Command) error {
	// raw only makes sense for a single file resource multiple objects aren't likely to do what you want.
	// the validator enforces this, so

	// 使用外部的apiserver
	if len(o.Raw) > 0 {
		restClient, err := f.RESTClient()
		if err != nil {
			return err
		}
		return rawhttp.RawPost(restClient, o.IOStreams, o.Raw, o.FilenameOptions.Filenames[0])
	}
	// 创建前 edit执行该函数
	if o.EditBeforeCreate {
		return RunEditOnCreate(f, o.PrintFlags, o.RecordFlags, o.IOStreams, cmd, &o.FilenameOptions, o.fieldManager)
	}
	schema, err := f.Validator(cmdutil.GetFlagBool(cmd, "validate"))
	if err != nil {
		return err
	}

	// 根据命令行获取ns
	cmdNamespace, enforceNamespace, err := f.ToRawKubeConfigLoader().Namespace()
	if err != nil {
		return err
	}
	// 构建builder对象
	r := f.NewBuilder().
		Unstructured().
		Schema(schema).
		ContinueOnError().
		NamespaceParam(cmdNamespace).DefaultNamespace().
		FilenameParam(enforceNamespace, &o.FilenameOptions).   // 读取配置文件
		LabelSelectorParam(o.Selector).
		Flatten().
		Do()
	err = r.Err()
	if err != nil {
		return err
	}

	count := 0
	// 调用visit函数创建资源，访问者模式
	err = r.Visit(func(info *resource.Info, err error) error {
		if err != nil {
			return err
		}
		if err := util.CreateOrUpdateAnnotation(cmdutil.GetFlagBool(cmd, cmdutil.ApplyAnnotationsFlag), info.Object, scheme.DefaultJSONEncoder()); err != nil {
			return cmdutil.AddSourceToErr("creating", info.Source, err)
		}

		if err := o.Recorder.Record(info.Object); err != nil {
			klog.V(4).Infof("error recording current command: %v", err)
		}

		if o.DryRunStrategy != cmdutil.DryRunClient {
			if o.DryRunStrategy == cmdutil.DryRunServer {
				if err := o.DryRunVerifier.HasSupport(info.Mapping.GroupVersionKind); err != nil {
					return cmdutil.AddSourceToErr("creating", info.Source, err)
				}
			}
			obj, err := resource.
				NewHelper(info.Client, info.Mapping).
				DryRun(o.DryRunStrategy == cmdutil.DryRunServer).
				WithFieldManager(o.fieldManager).
				Create(info.Namespace, true, info.Object)
			if err != nil {
				return cmdutil.AddSourceToErr("creating", info.Source, err)
			}
			info.Refresh(obj, true)
		}

		count++

		return o.PrintObj(info.Object)
	})
	if err != nil {
		return err
	}
	if count == 0 {
		return fmt.Errorf("no objects passed to create")
	}
	return nil
}
```

> 其中包含了：
>
> 构建`resourceBuilder`由`f.NewBuilder()`实现
>
> 调用visit方法创建资源由`r.Visit(func(info *resource.Info, err error)`实现

#### createCmd中的builder建造者设计模式

- 设计模式之建造者模式

  > 1. 建造者模式(Builder)模式：指将一个复杂对象的构造与它的表示分离
  > 2. 使同样的构建过程可以创建不同的对象，这样的设计模式被称为建造者模式
  > 3. 它是将一个复杂的对象分解为多个简单的对象，然后一步一步构建而成
  > 4. 它将变与不变相分离，即产品的组成部分是不变的，但每一部分是可以灵活选择的
  > 5. 更多用来针对复杂对象的创建
  >
  > **优点**：
  >
  > 1. 封装性好，构建和表示分离
  > 2. 扩展性好，各个具体的建造者相互独立，有利于系统的解耦
  > 3. 客户端不必知道产品内部组成的细节，建造者可以对创建过程逐步细化，而不对其他模块产生任何影响
  >
  > **缺点**：
  >
  > 1. 产品的组成部分必须相同，这限制了其使用范围
  > 2. 如果产品的内部变化复杂，如果产品内部发生变化，则建造者也要同步修改，后期维护成本较大
  >
  > **特点：**
  >
  > 1. 针对复杂对象的创建，字段非常多
  > 2. 开头的方法返回要创建对象的指针
  > 3. 所有的方法都返回的是建造对象的指针

  对应代码：

  ```go
  	r := f.NewBuilder().
  		Unstructured().
  		Schema(schema).
  		ContinueOnError().
  		NamespaceParam(cmdNamespace).DefaultNamespace().
  		FilenameParam(enforceNamespace, &o.FilenameOptions).   // 读取配置文件
  		LabelSelectorParam(o.Selector).
  		Flatten().
  		Do()
  ```

  其中Unstructured()

  ```go
  func (b *Builder) Unstructured() *Builder {
  	if b.mapper != nil {
  		b.errs = append(b.errs, fmt.Errorf("another mapper was already selected, cannot use unstructured types"))
  		return b
  	}
  	b.objectTyper = unstructuredscheme.NewUnstructuredObjectTyper()
  	b.mapper = &mapper{
  		localFn:      b.isLocal,
  		restMapperFn: b.restMapperFn,
  		clientFn:     b.getClient,
  		decoder:      &metadataValidatingDecoder{unstructured.UnstructuredJSONScheme},
  	}
  
  	return b
  }
  ```

  Schema

  ```go
  func (b *Builder) Schema(schema ContentValidator) *Builder {
  	b.schema = schema
  	return b
  }
  ```

  > 还有其他的，不一一展示了

  自定义案例展示：

  ```go
  type Person struct {
      Name string
      Age  int
      Sex  string
  }
  
  type PersonBuilder struct {
      person *Person
  }
  
  func NewPersonBuilder() *PersonBuilder {
      return &PersonBuilder{&Person{}}
  }
  
  func (b *PersonBuilder) SetName(name string) *PersonBuilder {
      b.person.Name = name
      return b
  }
  
  func (b *PersonBuilder) SetAge(age int) *PersonBuilder {
      b.person.Age = age
      return b
  }
  
  func (b *PersonBuilder) SetSex(sex string) *PersonBuilder {
      b.person.Sex = sex
      return b
  }
  
  func (b *PersonBuilder) Build() *Person {
      return b.person
  }
  func main() {
      person := NewPersonBuilder().
          SetName("Tom").
          SetAge(30).
          SetSex("Male").
          Build()
  
      fmt.Printf("%+v\n", *person)
  }
  
  
  ```

  

#### createCmd中的visitor访问者设计模式

- visitor访问者模式

  > 1. 访问者模式(Visitor Pattern)是一种将数据结构与数据操作分离的设计模式
  > 2. 指封装一些作用于某种数据结构中的各元素的造作
  > 3. 可以在不改变数据结构的前提下定义作用于这些元素的新的操作
  > 4. 属于行为型设计模式
  >
  > **应用场景：**
  >
  > 1. 数据结构稳定，作用于数据结构的操作经常变化的场景
  > 2. 需要数据结构与数据操作分离的场景
  > 3. 需要对不同数据类型(元素)进行操作，而不使用分支判断具体的类型场景
  >
  > **优点：**
  >
  > 1. 解耦了数据结构与数据操作，使得操作集合可以独立变化
  > 2. 可以通过扩展访问者角色，实现对数据集的不同操作，程序扩展性更好
  > 3. 元素具体类型并非单一，访问者均可操作
  > 4. 各角色职责分离，符合单一职责原则
  >
  > **缺点：**
  >
  > 1. 无法增加元素类型：若系统数据结构对象易于变化，经常有新的数据对象增加进来，则访问者类型必须增加对应元素类型的操作，违背了开闭原则
  > 2. 具体元素变更更加困难，具体元素增加属性，删除属性等操作会导致对应的访问者类型需要进行相应的修改，尤其当有大量访问者类型使，修改范围太大
  > 3. 违背依赖倒置原则，为了达到"区别对待"，访问者角色依赖的是具体元素类型，而不是抽象

#### kubectl create中通过Builder模式创建visitor并执行的过程

##### FilenameParam解析 -f 文件参数，创建一个visitor

- validate校验-f参数

  `vendor\k8s.io\kubectl\pkg\cmd\create\create.go`

  ```go
  	// 构建builder对象
  	r := f.NewBuilder().
  		Unstructured().
  		Schema(schema).
  		ContinueOnError().
  		NamespaceParam(cmdNamespace).DefaultNamespace().
  		FilenameParam(enforceNamespace, &o.FilenameOptions).   // 读取配置文件
  		LabelSelectorParam(o.Selector).
  		Flatten().
  		Do()
  ```

  其中FilenameParam对应的代码

  ```go
  func (b *Builder) FilenameParam(enforceNamespace bool, filenameOptions *FilenameOptions) *Builder {
  	if errs := filenameOptions.validate(); len(errs) > 0 {
  		b.errs = append(b.errs, errs...)
  		return b
  	}
  	recursive := filenameOptions.Recursive
  	paths := filenameOptions.Filenames
  	for _, s := range paths {
  		switch {
  		case s == "-":   // 如果是 - 表示接受标准输入 eg: cat test.yaml | kubectl create -
  			b.Stdin()
  		case strings.Index(s, "http://") == 0 || strings.Index(s, "https://") == 0:  // 源端的文件
  			url, err := url.Parse(s)
  			if err != nil {
  				b.errs = append(b.errs, fmt.Errorf("the URL passed to filename %q is not valid: %v", s, err))
  				continue
  			}
  			b.URL(defaultHttpGetAttempts, url)
  		default:   // 默认为文件
  			if !recursive {
  				b.singleItemImplied = true
  			}
  			b.Path(recursive, s)
  		}
  	}
  	if filenameOptions.Kustomize != "" {
  		b.paths = append(
  			b.paths,
  			&KustomizeVisitor{
  				mapper:  b.mapper,
  				dirPath: filenameOptions.Kustomize,
  				schema:  b.schema,
  				fSys:    filesys.MakeFsOnDisk(),
  			})
  	}
  
  	if enforceNamespace {
  		b.RequireNamespace()
  	}
  
  	return b
  }
  ```

  其中validate()对应的代码

  ```go
  // 参数校验
  func (o *FilenameOptions) validate() []error {
  	var errs []error
  	if len(o.Filenames) > 0 && len(o.Kustomize) > 0 {
  		errs = append(errs, fmt.Errorf("only one of -f or -k can be specified"))
  	}
  	if len(o.Kustomize) > 0 && o.Recursive {
  		errs = append(errs, fmt.Errorf("the -k flag can't be used with -f or -R"))
  	}
  	return errs
  }
  
  ```

  其中b.Path(recursive, s)的Path对应的代码

  ```go
  // 文件路径验证
  
  func (b *Builder) Path(recursive bool, paths ...string) *Builder {
  	for _, p := range paths {
  		_, err := os.Stat(p)
  		if os.IsNotExist(err) {
  			b.errs = append(b.errs, fmt.Errorf("the path %q does not exist", p))
  			continue
  		}
  		if err != nil {
  			b.errs = append(b.errs, fmt.Errorf("the path %q cannot be accessed: %v", p, err))
  			continue
  		}
  
  		visitors, err := ExpandPathsToFileVisitors(b.mapper, p, recursive, FileExtensions, b.schema)
  		if err != nil {
  			b.errs = append(b.errs, fmt.Errorf("error reading %q: %v", p, err))
  		}
  		if len(visitors) > 1 {
  			b.dir = true
  		}
  
  		b.paths = append(b.paths, visitors...)
  	}
  	if len(b.paths) == 0 && len(b.errs) == 0 {
  		b.errs = append(b.errs, fmt.Errorf("error reading %v: recognized file extensions are %v", paths, FileExtensions))
  	}
  	return b
  }
  ```

  Path调用ExpandPathsToFileVisitors生成visitor

  ```go
  func ExpandPathsToFileVisitors(mapper *mapper, paths string, recursive bool, extensions []string, schema ContentValidator) ([]Visitor, error) {
  	var visitors []Visitor
  	err := filepath.Walk(paths, func(path string, fi os.FileInfo, err error) error {
  		if err != nil {
  			return err
  		}
  
  		if fi.IsDir() {
  			if path != paths && !recursive {
  				return filepath.SkipDir
  			}
  			return nil
  		}
  		// Don't check extension if the filepath was passed explicitly
  		if path != paths && ignoreFile(path, extensions) {
  			return nil
  		}
  
  		visitor := &FileVisitor{
  			Path:          path,
  			StreamVisitor: NewStreamVisitor(nil, mapper, path, schema),
  		}
  
  		visitors = append(visitors, visitor)
  		return nil
  	})
  
  	if err != nil {
  		return nil, err
  	}
  	return visitors, nil
  }
  ```

  Do()创建一批visitor

  ```go
  func (b *Builder) Do() *Result {
  	r := b.visitorResult()
  	r.mapper = b.Mapper()
  	if r.err != nil {
  		return r
  	}
  	if b.flatten {
  		r.visitor = NewFlattenListVisitor(r.visitor, b.objectTyper, b.mapper)
  	}
  	helpers := []VisitorFunc{}
  	if b.defaultNamespace {
  		helpers = append(helpers, SetNamespace(b.namespace))
  	}
  	if b.requireNamespace {
  		helpers = append(helpers, RequireNamespace(b.namespace))
  	}
  	helpers = append(helpers, FilterNamespace)
  	if b.requireObject {
  		helpers = append(helpers, RetrieveLazy)
  	}
  	if b.continueOnError {
  		r.visitor = NewDecoratedVisitor(ContinueOnErrorVisitor{r.visitor}, helpers...)
  	} else {
  		r.visitor = NewDecoratedVisitor(r.visitor, helpers...)
  	}
  	return r
  }
  ```

  > 其返回值*Result，为：
  >
  > ```go
  > type Result struct {
  >    err     error
  >    visitor Visitor
  > 
  >    sources            []Visitor
  >    singleItemImplied  bool
  >    targetsSingleItems bool
  > 
  >    mapper       *mapper
  >    ignoreErrors []utilerrors.Matcher
  > 
  >    // populated by a call to Infos
  >    info []*Info
  > }
  > ```
  >
  > 其中helpers代表一笔visitorFunc，可以查看一下`RequireNamespace`
  >
  > ```go
  > func RequireNamespace(namespace string) VisitorFunc {
  > 	return func(info *Info, err error) error {
  > 		if err != nil {
  > 			return err
  > 		}
  > 		if !info.Namespaced() {
  > 			return nil
  > 		}
  > 		if len(info.Namespace) == 0 {
  > 			info.Namespace = namespace
  > 			UpdateObjectNamespace(info, nil)
  > 			return nil
  > 		}
  > 		if info.Namespace != namespace {
  > 			return fmt.Errorf("the namespace from the provided object %q does not match the namespace %q. You must pass '--namespace=%s' to perform this operation.", info.Namespace, namespace, info.Namespace)
  > 		}
  > 		return nil
  > 	}
  > }
  > ```
  >
  > 

  其对应到k8s中 使用kubectl进行验证

  ```bash
  # kubectl create -f jenkins-deployment.yaml  -k jenkins-pvc.yaml  
  error: only one of -f or -k can be specified
  
  # kubectl create   -k jenkins-pvc.yaml  -R
  error: the -k flag can't be used with -f or -R
  
  # kubectl create   -f  jenkins-pvc.yaml1 
  error: the path "jenkins-pvc.yaml1" does not exist
  
  
  ```

  



