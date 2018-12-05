# 问题的现象的描述

凡是在Java类的中文被打印输出时变为乱码。

# 问题发生前后的环境差异检查

项目的打包是通过Jenkins进行的，
由于对Jenkins进行了一次升级和数据迁移，
而乱码问题是发生在升级迁移之后，所以排查
问题从前后打包环境的差异性开始


## 发生问题之前的打包环境

[root@server ~]# java -version
java -version
java version "1.8.0_152"
Java(TM) SE Runtime Environment (build 1.8.0_152-b16)
Java HotSpot(TM) 64-Bit Server VM (build 25.152-b16, mixed mode)


Jenkins版本为2.53，启动方式为


/etc/init.d/jenkins start


# 发生问题时的打包环境


jenkins@84d299b30e0d:/$ java -version
java -version
openjdk version "1.8.0_171"
OpenJDK Runtime Environment (build 1.8.0_171-8u171-b11-1~deb9u1-b11)
OpenJDK 64-Bit Server VM (build 25.171-b11, mixed mode)


Jenkins版本2.107.3，启动方式为


docker run -d --name myjenkins -p 80:8080 -p 50000:50000 -v /some/path/jenkins_data:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkins


## 部署的服务器信息

[sshuser@server ~]$ rpm -q centos-release
centos-release-7-5.1804.el7.centos.x86_64

## 部署的服务器的编码信息

[sshuser@server ~]$ locale
LANG=en_US.UTF-8
LC_CTYPE="en_US.UTF-8"
LC_NUMERIC=zh_CN.UTF-8
LC_TIME=zh_CN.UTF-8
LC_COLLATE="en_US.UTF-8"
LC_MONETARY=zh_CN.UTF-8
LC_MESSAGES="en_US.UTF-8"
LC_PAPER=zh_CN.UTF-8
LC_NAME="en_US.UTF-8"
LC_ADDRESS="en_US.UTF-8"
LC_TELEPHONE="en_US.UTF-8"
LC_MEASUREMENT=zh_CN.UTF-8
LC_IDENTIFICATION="en_US.UTF-8"
LC_ALL=


## 部署的服务器上Java的编码信息

[root@srv ~]# java -XshowSettings:all -version
VM settings:
    Max. Heap Size (Estimated): 1.70G
    Ergonomics Machine Class: server
    Using VM: OpenJDK 64-Bit Server VM

Property settings:
    awt.toolkit = sun.awt.X11.XToolkit
    file.encoding = UTF-8
    file.encoding.pkg = sun.io
    file.separator = /
    java.awt.graphicsenv = sun.awt.X11GraphicsEnvironment
    java.awt.printerjob = sun.print.PSPrinterJob
    java.class.path = .
    java.class.version = 52.0
    java.endorsed.dirs = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64/jre/lib/endorsed
    java.ext.dirs = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64/jre/lib/ext
        /usr/java/packages/lib/ext
    java.home = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64/jre
    java.io.tmpdir = /tmp
    java.library.path = /usr/java/packages/lib/amd64
        /usr/lib64
        /lib64
        /lib
        /usr/lib
    java.runtime.name = OpenJDK Runtime Environment
    java.runtime.version = 1.8.0_161-b14
    java.specification.name = Java Platform API Specification
    java.specification.vendor = Oracle Corporation
    java.specification.version = 1.8
    java.vendor = Oracle Corporation
    java.vendor.url = http://java.oracle.com/
    java.vendor.url.bug = http://bugreport.sun.com/bugreport/
    java.version = 1.8.0_161
    java.vm.info = mixed mode
    java.vm.name = OpenJDK 64-Bit Server VM
    java.vm.specification.name = Java Virtual Machine Specification
    java.vm.specification.vendor = Oracle Corporation
    java.vm.specification.version = 1.8
    java.vm.vendor = Oracle Corporation
    java.vm.version = 25.161-b14
    line.separator = \n
    os.arch = amd64
    os.name = Linux
    os.version = 3.10.0-862.el7.x86_64
    path.separator = :
    sun.arch.data.model = 64
    sun.boot.class.path = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64/jre/lib/resources.jar
        /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64/jre/lib/rt.jar
        /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64/jre/lib/sunrsasign.jar
        /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64/jre/lib/jsse.jar
        /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64/jre/lib/jce.jar
        /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64/jre/lib/charsets.jar
        /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64/jre/lib/jfr.jar
        /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64/jre/classes
    sun.boot.library.path = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64/jre/lib/amd64
    sun.cpu.endian = little
    sun.cpu.isalist =
    sun.io.unicode.encoding = UnicodeLittle
    sun.java.launcher = SUN_STANDARD
    sun.jnu.encoding = UTF-8
    sun.management.compiler = HotSpot 64-Bit Tiered Compilers
    sun.os.patch.level = unknown
    user.country = CN
    user.dir = /root
    user.home = /root
    user.language = zh
    user.name = root
    user.timezone =

Locale settings:
    default locale = ÖÐÎÄ
    default display locale = ÖÐÎÄ (ÖÐ¹ú)
    default format locale = ÖÐÎÄ (ÖÐ¹ú)
    available locales = , ar, ar_AE, ar_BH, ar_DZ, ar_EG, ar_IQ, ar_JO,
        ar_KW, ar_LB, ar_LY, ar_MA, ar_OM, ar_QA, ar_SA, ar_SD,
        ar_SY, ar_TN, ar_YE, be, be_BY, bg, bg_BG, ca,
        ca_ES, cs, cs_CZ, da, da_DK, de, de_AT, de_CH,
        de_DE, de_GR, de_LU, el, el_CY, el_GR, en, en_AU,
        en_CA, en_GB, en_IE, en_IN, en_MT, en_NZ, en_PH, en_SG,
        en_US, en_ZA, es, es_AR, es_BO, es_CL, es_CO, es_CR,
        es_CU, es_DO, es_EC, es_ES, es_GT, es_HN, es_MX, es_NI,
        es_PA, es_PE, es_PR, es_PY, es_SV, es_US, es_UY, es_VE,
        et, et_EE, fi, fi_FI, fr, fr_BE, fr_CA, fr_CH,
        fr_FR, fr_LU, ga, ga_IE, hi, hi_IN, hr, hr_HR,
        hu, hu_HU, in, in_ID, is, is_IS, it, it_CH,
        it_IT, iw, iw_IL, ja, ja_JP, ja_JP_JP_#u-ca-japanese, ko, ko_KR,
        lt, lt_LT, lv, lv_LV, mk, mk_MK, ms, ms_MY,
        mt, mt_MT, nl, nl_BE, nl_NL, no, no_NO, no_NO_NY,
        pl, pl_PL, pt, pt_BR, pt_PT, ro, ro_RO, ru,
        ru_RU, sk, sk_SK, sl, sl_SI, sq, sq_AL, sr,
        sr_BA, sr_BA_#Latn, sr_CS, sr_ME, sr_ME_#Latn, sr_RS, sr_RS_#Latn, sr__#Latn,
        sv, sv_SE, th, th_TH, th_TH_TH_#u-nu-thai, tr, tr_TR, uk,
        uk_UA, vi, vi_VN, zh, zh_CN, zh_HK, zh_SG, zh_TW

openjdk version "1.8.0_161"
OpenJDK Runtime Environment (build 1.8.0_161-b14)
OpenJDK 64-Bit Server VM (build 25.161-b14, mixed mode)

# 问题分析和解决过程

Java文件中的中文输出到日志变为乱码，
猜测可能和Javac编译时没有指定UTF-8有关系，
项目是用Maven构建的，检查Maven的编码配置信息如下

       <maven.compiler.source>1.8</maven.compiler.source>
       <maven.compiler.target>1.8</maven.compiler.target>
       <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

所以排除和Maven的关系。

从服务器上的Java编码情况来看

    sun.jnu.encoding = UTF-8
    file.encoding = UTF-8

运行时默认的编码设置都是UTF-8。
那么编译时和运行时的编码设置都是UTF-8，理论上不应该出
Java文件中的中文输出为乱码，所以怀疑是应用运行时没有获取到Java的默认编码设置，
于是尝试在启动的脚本中增加时指定参数： -Dfile.encoding=UTF-8，再观察应用的中文日志输出，没有出现乱码。

# 结论
根据解决的过程分析应该是应用在运行时，没有获取到系统的编码，通过手动指定jvm启动
编码参数解决了该问题，虽然这个问题解决了，但是还需要深入调查，确认是由于Jenkins版本的差异导致的，
还是由于编译时JDK的差异导致，根据Jenkins编译时也是依赖系统jdk的java命令启动jvm，因此推测问题
的根本原因是Open JDK和Oracle JDK差异导致的问题，可能是Open JDK没有正确的获取到编UTF-8编码引起，
这个问题的本质原因在后续的文章中分析
