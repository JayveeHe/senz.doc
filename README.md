Senz CI 帮助文档
===

_@Authored by: Woodie_

_@Updated at: Thursday, June 11, 2015_

工作环境
---
- Jenkins 持续集成系统工具
- LeanCloud 测试和生产部署环境
- DaoCloud 测试和生产部署环境
- Github 代码库以及版本控制工具

基本概念
---
### Jenkins
一个持续集成环境需要包括三个方面要素：代码存储库、构建过程和持续集成服务器。这里代码存储库我们采用的*Github*，持续集成服务器是Senz项目下的*AliYun 服务器*,你可以登陆Senz项目[Jenkins管理端][]来查看和管理当前的*Jenkins Job*（每一个job都是一连串的交互操作，需要在管理端进行相应的配置，以实现项目自动化测试和部署），构建过程即为部署在持续集成服务器上的*开源工具 Jenkins*。
Jenkins提供了一种易于使用的持续集成系统，使开发者从繁杂的集成中解脱出来，专注于更为重要的业务逻辑实现上。同时 Jenkins 能实时监控集成中存在的错误，提供详细的日志文件和提醒功能，还能用图表的形式形象地展示项目构建的趋势和稳定性。

### LeanCloud
[LeanCloud][]是国内针对移动应用的一站式*BaaS*云端服务。在可见的未来，我们大部分的*NodeJS后端服务*和部分独立的*Python（django或者flask）后端服务*都会部署在LeanCloud环境下。LeanCloud为我们的移动应用产品提供完整的数据存储、代码测试环境、生产环境（docker容器）等服务，并且提供了一键式的deploy和publish项目API。
[LeanCloud]: https://leancloud.cn/

### DaoCloud
[DaoCloud][]是国内首个*Docker Hub镜像服务*，提供互联网应用的持续集成、镜像构建、发布管理、容器托管解决方案。
目前，我们所有大计算量的算法子服务均会部署在DaoCloud环境上。DaoCloud的优点是界面友好，操作简单，容易上手，因此即使是不懂docker的新手来操作也能分分钟搞定。
[DaoCloud]: https://www.daocloud.io/

整体流程
---
任何项目对应一个github repo，每个repo目前有两个branch，分别是dev和master，每个branch需要维护一个独立的App，分别用于开发环境（dev branch）和生产环境（master branch）。
***注意*** 开发和生产环境是两个完全独立的App，各自拥有自己的environment、db以及不同的代码branch（属于同一个repo）。
在部署好了环境和CI流程后，一个项目从开发到上线至生产环境的大致流程如下：
- 在dev branch上进行日常的项目开发，开发了一个新的feature后，拉取代码到___本地环境___进行***Mock Server测试***；
- git push origin dev会触发开发环境下代码的Mock Server测试环节；
- 代码调试无误，通过Mock Server测试后，发布到___开发环境___，在模仿真实环境的网络环境中进行***线上测试***；
- git tag dev并git push后，会触发线上测试环节;
- 当积累了一定量的feature，并且在开发环境上运行一段时间没有问题，可以发布到___生产环境___，进行***生产部署***（***注：生产环境部署会引起短时间内的生产环境宕机，当要执行该操作时，必须先由senz项目负责人统一确认各个子服务情况后，择机部署。必要时，需通过邮件或者官网告知用户***）；
- 你也可以在生产部署前进行一次Mock Server测试，以确保代码无误；
- git push origin master会触发生产环境下代码的Mock Server测试环节；
- git tag prod并git push后，会触发生产部署环节；

### git tag发布流程(tag不能相同,)
- 开发环境：git tag dev-[version],git push origin dev-[version]
- 生产环境：git tag prod-[version],git push origin dev-[version]
- leancloud会通过jenkins识别github tag行为。daocloud能自主识别github tag行为。两者识别后的行为都是发布app到指定的相应环境中。

### NodeJS后端服务（数据流操作）

- 在github上***创建代码库***，该repo有两个branch，分别是master和dev，对应不同的环境。每当在某个环境中（开发环境或生环境）需要进行Mock Server测试时，使用git push branch_name；每当在某个环境需要发布最新的release版本时，使用git tag + 版本号（版本号统一使用vX.X.X的形式）；项目总需要先在开发环境下进行测试运行后，再在生产环境下进行测试和运行；
- 使用LeanCloud提供的最新***LeanEngine***环境开发；
- 项目中需要加入***判断工作环境***代码，利用***环境变量***来区分现在是*生产环境*、*开发环境*还是*本地环境*（目前暂未用到，但是需要保留该关键字），不同的工作环境是完全不同的应用，从不同的branch提取代码、运行在不同的主机环境、使用不同的log系统（即不同的第三方log工具token），不同的数据库（详见下文“生产环境和开发环境”章节）。
- 开发项目用github做代码库和版本控制（详见下文“Jenkins CI”章节），rollbar和logentries做错误处理和日志记录（详见下文“项目日志和异常处理”章节），express的supertest做单元测试（详见下文“单元测试”章节）；
- 在LeanCloud上创建两个项目，分别命名为senz.xxx.xxx（生产环境）和dev_senz.xxx.xxx（开发环境），并分别两个项目[配置git部署][]，不同环境对应不同的git branch（生产环境对应master，开发环境对应dev）
- 在[Jenkins管理端][]***创建testJob***，每当代码库git push到某branch时，自动触发该环境下项目test事件：
    + 在部署云主机（Aliyun）创建NodeJS运行环境容器，
    + 在容器中执行Mock Server test脚本，报告测试结果（详见下文“Jenkins CI”章节）；
- 在[Jenkins管理端][]***创建publishJob***，每当git tag dev or prod发布某个branch的最新release版本到对应环境时，自动触发该环境项目publish事件：
    + 在部署云主（Aliyun）上执行部署命令：
    + 启动项目部署到对应环境中（详见下文“Jenkins CI”章节）。
[Jenkins管理端]: http://182.92.72.69:8080/
[配置git部署]: https://leancloud.cn/docs/leanengine_guide-node.html#部署

### Python后端服务（算法模块）

- 在github上***创建代码库***，该repo有两个branch，分别是master和dev，对应不同的环境。每当在某个环境中（开发环境或生环境）需要进行Mock Server测试时，使用git push branch_name；每当在某个环境需要发布最新的release版本时，使用git tag + 版本号（版本号统一使用vX.X.X的形式）；项目总需要先在开发环境下进行测试运行后，再在生产环境下进行测试和运行；
- 使用***flask***进行开发；
- 项目中需要加入***判断工作环境***代码，利用***环境变量***来区分现在是*生产环境*、*开发环境*还是*本地环境*（目前暂未用到，但是需要保留该关键字），不同的工作环境是完全不同的应用，从不同的branch提取代码、运行在不同的主机环境、使用不同的log系统（即不同的第三方log工具token），不同的数据库（详见下文“生产环境和开发环境”章节）。
- 开发项目用github做代码库和版本控制（详见下文“Jenkins CI”章节），rollbar和logentries做错误处理和日志记录（详见下文“项目日志和异常处理”章节），flask的flask-test做mock server测试（详见下文“单元测试”章节）；
- 编写配置flask和所需依赖环境的***Dockerfile***；（具体可以参照[这个项目][]的Dockerfile，或者想深入了解如何编写Dockerfile可以参考[docker book][]）；
- 在***DaoCloud***上创建两个项目，分别对应开发环境和测试环境，注意：每次构建选择手动构建，不同环境下的项目对应选择不同的branch（生产环境对应master，开发环境对应dev），因为自动构建会默认从master提取代码。其他配置流程和参数保持一致（目前daocloud会在短期内改为自动构建时可以指定分支）。
- 在项目根目录编写***DaoCloud.yml***文件，用于指导DaoCloud进行自动化测试，具体参见[DaoCloud.yml文档][]和[DaoCloud.yml示例][]；（详见下文“DaoCloud CI”章节）

[这个项目]: https://github.com/petchat/senz.template.docker.flask/blob/master/flask_app/test.py
[docker book]: http://yeasy.gitbooks.io/docker_practice/content/
[DaoCloud.yml文档]: https://github.com/DaoCloud/daocloud-doc/blob/master/DaoCloudCI.md
[DaoCloud.yml示例]: https://github.com/DaoCloud?utf8=%E2%9C%93&query=sample

生产环境和开发环境
----
我们需要区分代码的工作环境，主要是依靠识别环境变量，我们定义```APP_ENV```为识别工作环境的环境变量名。每次程序启动时都会读取本地名为“APP_ENV”的环境变量值，该环境变量有三种取值：“dev”、“prod”和“local”分别代表开发环境、生产环境和本地环境。

### 相关配置
不同的工作环境需要有不同的配置，主要包括：
- 数据库：目前我们所有的数据库都是依托在LeanCloud平台上，开发环境数据库为dev@开头。后面名称开发和生产一致。
- log系统：目前我们使用了logentries和rollbar两个第三方trace服务，因此不同的工作环境对应不同的logentries和rollbar的token即可。
- 不同的容器：对于在daocloud上的项目而言，需要为不同的工作环境配置不同的容器。（详见下文“DaoCloud CI”章节）

### 数据库备份
由于每个项目都需要维护两套同样的数据库，因此需要自己为自己的项目编写一个脚本，来定期同步两个数据库中的所有数据，日后我们可能会提供这样的脚本接口供你使用。备份代码示例见冯老师github项目。

### Example
我们有一个完整的项目以供参考。这是一个flask项目，用来提供poi概率计算服务，你可以参考该项目的[根据环境变量配置token][]例子来进一步理解。
[根据环境变量配置token]: https://github.com/petchat/senz.middleware.poi.poiprob/blob/master/flask_app/poi_analyser_lib/config.py

项目日志和异常处理
---
任何开发项目都需要对运行代码的重要输出和错误异常进行记录，我们采用logentries+rollbar组合方案来实现项目的日志记录和异常处理工作。
- ***Rollbar*** 功能强大，可以用于记录代码输出，上传捕获和未捕获到的异常到云端。针对我们的需求，我们仅使用rollbar上传*未捕获到的异常*（Uncaught exception）的feature，本质上rollbar在项目框架上深度定制了一个middleware，以此来捕获哪些我们没有catch到的exception。可以查看[rollbar文档][]来深入了解。
- ***Logentries*** 主要用于记录代码中的各种输出，并同样上传到服务器，提供统一友好的用户UI方便开发者浏览查找。其优势在于对输出信息进行细致的分级记录，分别包括info、warning、debug、error四个等级的日志记录类型，并且提供丰富的查询接口来方便开发者快速定位到希望查看的对应日志信息。可以查看[logentries文档][]来深入了解。
- 目前在flask项目中rollbar和logentries之间有一定的兼容性问题，主要体现在二者都使用了python内建的logger模块来进行log消息，导致两个模块分别向自己的服务器发送log记录时出现冲突，目前解决冲突的唯一方法是***请求flask项目的http header中不加content-type字段***。该issue已经发至rollbar support。具体进展请咨询张先生。
[rollbar文档]:https://rollbar.com/docs/
[logentries文档]:https://logentries.com/doc/

### 日志类型
对于项目代码的输出信息我们规定包括四个级别，分别是：INFO，DEBUG，WARNING，ERROR。每个级别对应的输出信息应该是：
- INFO： 代码关键环节的信息，以```[模块名] 输出信息```的格式输出；
- DEBUG：需要输出的重要参数信息，以```[模块名] 参数名: 参数内容```的格式输出；
- WARNING：出现了不合法的代码环节处，但是并不影响代码的执行（可能原因例如：前后版本不一致，导致输入参数有所区别但是不会致错），以```[模块名] 警告信息```的格式输出；
- ERROR：代码抛出异常，可能会造成执行失败，该错误一般为已知错误类型，或自己定义的错误，以```[错误类型] 错误信息. TRACEBACK: traceback记录```的格式输出。
- log 规范请参考 [log 规范@trello][https://trello.com/c/S9DIL8Lb]

### 自定义错误
为了方便日后在数据量庞大的生产环境中快速定位到我们需要查找的错误，或者对错误信息进行统计分析。我们需要简单的定义一些自己的错误类型，一般是以一个exception.py或者exception.js的形式。
- 定义错误类型原则： 
    + 只对该项目中***核心处理模块***定义自己的错误，而例如输入参数不合法等数据整理、准备过程中出现的一些常见错误直接catch住并抛出自己定义的错误即可；
    + 核心处理模块中出现的任何可能的错误都尽可能merge到自己定义的错误中，保证核心模块只会抛出自己定义的错误，这样一旦核心模块出问题我们就能很很清楚的知道错误类型的范围；
    + 自己定义的错误类型不宜太多太复杂，基本描述清楚核心模块可能出错的几个环节即可。
- 如何反馈错误：
    + 对于我们已知的错误，无论是常见系统定义的错误还是我们自己定义的错误，都由logentries.error汇报到logentries服务端。
    + 对于我们未知的错误，即我们没有catch住得错误，都由rollbar汇报到rollbar服务端（In fact，rollbar只用来负责汇报哪些我们无法catch到的未知错误，具体实现原理见本章“概述”）
- 输出错误信息的要求：
    + exception文件中需要定义一个exception基类（这个基类继承自系统内建的exception），其他所有其他的exception类型都继承自这个基类，该基类主要负责收取当前出错时的traceback。
    + 对于一个自定义exception，首先需要继承自己定义的exception基类，然后该exception需要有接口能返回当前错误收取到的一些基本信息，例如python项目中可以用
    ```python
    def __str__(self):
        return "Exception info"
    ```
    来返回错误内容。

### Example
我们有一个完整的项目以供参考。这是一个flask项目，用来提供poi概率计算服务，你可以参考该项目的[exception定义][],以及如何在[核心模块中抛出自定义异常][]，如何在[主程序中抛出常见错误和自定义错误][]。
[exception定义]: https://github.com/petchat/senz.middleware.poi.poiprob/blob/master/flask_app/poi_analyser_lib/exception.py
[核心模块中抛出自定义异常]: https://github.com/petchat/senz.middleware.poi.poiprob/blob/master/flask_app/poi_analyser_lib/predictor.py
[主程序中抛出常见错误和自定义错误]: https://github.com/petchat/senz.middleware.poi.poiprob/blob/master/flask_app/app.py

Mock Server测试
---
任何项目在上线进入生产环境前都需要进行不同程度的Mock Server测试，保证代码在各个环节都正常运行后才能投入使用。目前python flask项目采用flask自带的unittest模块进行Mock Server测试，而NodeJS LeanCloud项目下采用express框架下的supertest。

- [flask项目 unittests文档][] 以及[unittests示例][]
- [LeanCloud项目 supertest文档][] 以及[supertest示例][]

[flask项目 unittests文档]: http://flask.pocoo.org/docs/0.10/testing/
[LeanCloud项目 supertest文档]: http://www.scotchmedia.com/tutorials/express/authentication/2/02
[unittests示例]: https://github.com/petchat/senz.middleware.poi.poiprob/blob/master/flask_app/test/test_app.py
[supertest示例]: https://github.com/petchat/senz.middleware.rabbitmq.type/blob/master/cloud/test.js

Jenkins CI
---
下面介绍一下Jenkins里的相关操作和概念，以及在代码管理上的一些建议。Enjoy it！
根据不同项目需求，我们暂定：
- LeanCloud项目需要创建两个实际的项目分别用于开发环境和生产环境，每个环境对应一个testJob和一个publishJob(包含两个Jenkins Jobs），因此总共会有6个Jenkins Jobs（两个纯test job对应git push，一个test job和一个publish job对应git tag dev*，一个test job和一个publish job对应git tag prod*）；
- Flask项目暂时不用在Jenkins中进行管理，所有的CI工作都在DaoCloud环境下进行。

###github webhook添加
github要添加webhook service才能在jenkins勾选Build when a change is pushed to GitHub的条件下，通过git push触发。
- public & private项目添加方法：在github settings里的Webhooks&services里点击 Add service。选择Jenkins（Github plugin）。然后再Jenkins hook url里填写 http://182.92.72.69:8080/github-webhook/ 。并点击add services。


### 如何创建testJob
首先需要登录到我们的Senz Jenkins管理端，账号和密码见trello的[Account Card][]

- testJob本质上是一个Jenkins Job，登录后首先点击左上角的***New Item***，来创建一个新的Jenkins Job；
- 输入Item name，以格式
    ```
    senz.xxx.xxx_Test or dev_senz.xxx.xxx_Test
    ```
  选择***freestyle project***；
- 进入configure页面，项目名即为刚刚设定的Item name；
- 输入github对应代码库url，来指定项目代码库；
- 勾选Restrict where this project can be run，以限定该job最终运行环境为我们指定的机器，因为是testJob，所以部署在我们的aliyun 1服务器上，Label Expression内填***python_main_server***。你可以在主页左下方上查看Senz项目的主机情况，每个主机都有一个对应的label；
- ***Source Code Management***选择Git，并在Repository url里填写构建的project url（例如，https://github.com/petchat/petchat.app.yochat.cloud），Branch Specifier内根据对应的项目环境填写*/master or */dev，以指定只检查相应branch下的变化。####注：private项目需要添加Credentials。在Credentials里选择bboalimoe开头的key。如果想定义自己的，则点击旁边Add button。在text框中输入自己github的username和password并保存即可。
- ***Build Triggers***选择Build when a change is pushed to GitHub，每次对应 branch上的代码发生变化时触发build下的操作；
- ***Build下的Excute shell***中填写执行测试用例的shell脚本，例如：

    ```##shell
    pip install -r ./flask_app/requirements.txt
    nosetests ./flask_app/test.py 
    ```
    
- 点击save和apply来保存和生效配置信息
- 配置完成后如果你想立即运行，可以点击左侧的***Build Now***来立即执行一遍生效的Job，同时在console output查看服务器的运行输出

[Account Card]: https://trello.com/c/WKkAoaYS

### 如何创建publishJob
publishJob本质上是两个JenkinsJob，第一个Job和testJob几乎一样，除了：
- 项目名为：
    ```
    senz.xxx.xxx_Pretest 
    ```；
- ***Branch Specifier***内填写refs/tags/dev*（开发环境）refs/tags/prod*，以指定一旦有新的tag产生则触发build操作；
- 以及***Post-build Actions***的Build other projects中Projects to build填写第二个Job的项目名，并选择Trigger only if build is stable，以指定该Job完成后执行Publish Job。(jenkins和github触发机制待确定。)

第二个Job用来在生产环境部署代码，操作也很简单，主要区别如下：

- 第二个项目名称为：
    ```
    senz.xxx.xxx_Publish
    ```
- 指定项目的代码库，和master branch
- ***Restrict where this project can be run***和***Build Triggers***均不用选择。需要说明的是publishJob的build操作仅用向LeanCloud发送很轻量的HTTP请求即可，因此不用指定具体哪一台机器来执行这个Job，其次；而build的触发由上一个pretestJob来触发，因此不由其他触发源触发，因此也不用特殊指定。
- ***Build下的Excute shell***里填写avoscloud部署命令，并指定使用的branch。（部署主机Aliyun上提前安装了avoscloud工具，如何使用这两个命令进行部署，详细可参见[git仓库部署][]）

    ```shell
    avoscloud deploy
    avoscloud publish
    ```
- ##注：private的github项目需要在
    
[git仓库部署]: https://leancloud.cn/docs/cloud_code_commandline.html#Git仓库部署

DaoCloud CI
---
DaoCloud主要分为两部分CI工作，分别是代码构建和持续集成

### 大致流程
- 代码构建：
    + 由git tag触发
    + 从配置的git repo上拉取最新的master branch代码
    + 根据项目工程根目录下的Dockerfile指导build docker image
    + run docker container（启动容器）
- 持续集成：
    + 由git push触发
    + 从配置的git repo上拉取最新的master branch代码
    + 根据项目工程根目录下的daocloud.yml中指定的image，来指导build docker image
    + run docker container（启动容器）
    + 配置daocloud.yml中env下的环境变量
    + run daocloud.yml中install下的命令（在这里配置所需环境）
    + run daocloud.yml中before_script下的命令（在这里输出环境变量值）
    + run daocloud.yml中script下的命令（在这里执行nosetests）

### 操作注意事项
具体如何创建DaoCloud项目可以参看DaoCloud官方文档，操作流程很简单，这里只提及几点需要注意的地方：
- 每次构建的时候需要选择***手动构建***，而***不是自动构建***，因为手动构建可以选择具体的branch；
- 每个环境下需要对应创建一个DaoCloud项目。

Github代码管理
---
我们推荐任何一个开发项目都持有两个branch，分别是***master***和***dev***。
* 日常的项目开发和bug调试都在dev下进行，当开发出了一个新feature或者到达某个可以运行的阶段，可以push到dev分支上，触发测试流程；当需要在开发环境上进行实际运行测试，可以git tag dev，在开发环境上稳定运行一段时后，再merge到master branch上。
* 同样的，每当项目新feature能在开发环境稳定运行后，需要发布release版，合并到master branch，触发生产环境上的测试流程，测试通过后git tag prod_vX.X.X（格式确定一个），正式发布到生产环境运行。
