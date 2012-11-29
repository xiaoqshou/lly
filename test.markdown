#Linux 文件系统权限机制分析

##Introduction

  * 文件/目录访问控制是Linux操作系统安全的重要组成部分。传统的Linux操作系统支持用户-用户组-其它用户的访问控制机制，来限定系统用户对文件/目录的访问权限，该机制已经广泛为用户所接受和应用。而在实际的使用过程中，用户意识到在很多应用场景该机制并不能灵活、高效地满足访问控制需求，因而自Linux内核2.6版本开始便支持更为灵活的ACL(访问控制列表)机制。本文将通过实例来详细介绍这两种机制的原理及使用。

##概要描述
    Linux文件权限分为普通权限和ACL权限，传统的linux文件系统采用的是一种基于普通权限的自主访问控制机制
	* 1、传统的用户-用户组-其他用户(UGO)访问控制机制

　　UGO(user，group，other)模式原理

　　Linux系统中的每个文件和目录都有访问许可权限，通过其确定谁可以通过何种方式对文件和目录进行访问和操作。文件或目录的访问权限分为只读，只写和可执行三种。以文件为例，只读权限表示只允许读其内容，而禁止对其做任何的更改操作;只写权限允许对文件进行任何的修改操作;可执行权限表示允许将该文件作为一个程序执行。文件被创建时，文件所有者自动拥有对该文件的读、写和可执行权限，以便于对文件的阅读和修改。用户也可根据需要把访问权限设置为需要的任何组合。

　　有三种不同类型的用户可对文件或目录进行访问：文件所有者，同组用户、其他用户。所有者一般是文件的创建者。它可以允许同组用户有权访问文件，还可以将文件的访问权限赋予系统中的其他用户。在这种情况下，系统中的每一位用户都能访问该用户拥有的文件或目录。

　　每一个文件或目录的访问权限都有三组，每组用三位表示，分别为文件属主的读、写和执行权限;与属主同组的用户的读、写和执行权限;系统中其他用户的读、写和执行权限。当用ls-l命令显示文件或目录的详细信息时，最左边的一列为文件的访问权限。例如：

　　#ls-l

　　总计76

　　-rw-------1rootroot79711-0620:41anaconda-ks.cfg

　　drwxr-xr-x2rootroot409611-0613:50Desktop

　　-rw-r--r--1rootroot4484311-0620:40install.log

　　-rw-r--r--1rootroot751311-0620:35install.log.syslog

　　横线代表空许可(即表示不具有该权限)。r代表只读，w代表写，x代表可执行。注意：这里共有10个位置。第1个字符指定了文件类型。在通常意义上，一个目录也是一个文件。如果第1个字符是横线，表示是一个非目录的文件。如果是d，表示是一个目录。后面的9个字符每三个构成一组，依次表示文件主、组用户、其他用户对该文件的访问权限。

　　确定了一个文件的访问权限后，用户可以利用Linux系统提供的chmod命令来重新设定不同的访问权限。也可以利用chown命令来更改某个文件或目录的所有者。

　　2、扩展的访问控制列表(ACL)方式


　　* 为什么要采用ACL

　　UGO访问控制机制在很多情况下难以满足实际文件/目录访问授权的需求，比如，要设定一个组中的部分用户对特定的文件/目录具有读取和访问权限(rw-)，而另外一部分用户只能具备读权限(r--);这在传统的Linux访问控制中无法通过单纯地建立新的组和用户来实现。因此，为了解决这些问题，人们提出了一种新的访问控制方法，也就是访问控制列表(ACL，AccessControlList)。

　　ACL是一个POSIX(可移植操作系统接口，PortableOperatingSystemInterface)标准。目前，支持ACL需要内核和文件系统的支持。现在2.6内核配合EXT2/EXT3,JFS,XFS,ReiserFS等文件系统都是可以支持ACL的。在目前主流的发行套件，如RedHatEnterpriseLinux(RHEL)5、RHEL6、Fedora16等等，都已经支持ACL。

    * ACL简介
　　ACL的类型及权限位

　　ACL信息是以对象为索引，把所有对某一对象拥有访问权限的用户及其访问权限都组合在一起与此对象关联，称为这一对象的ACL属性。如果要对某一访问请求进行判断，则首先要找到被访问的对象，然后再在此对象的ACL属性中查找有无此请求所包含的用户，最后在此用户的访问权限集中检查有无此请求所需要的访问权限，如果有此权限则此请求合法。
　　ACL属性由一系列访问控制项组正，每一项为一个三元组：{权限类型 用户标志 权限集}，它代表来某个访问者对此对象的访问权限。ACL除了对传统的三种访问者身份的描述外，海天加来对另外的用户和组的描述。
　　ACL是由一系列的AccessEntry所组成的。每一条AccessEntry定义了特定的类别可以对文件拥有的操作权限。AccessEntry主要包括6个，可分为两大类：一类包括owner、owninggroup和other，对应传统UGO机制中的user、group和other;一类则包括nameduser、namedgroup和mask。这六类的主要说明如下：

　　user：相当于Linux里文件所有者的permission

　　nameduser：定义了额外的用户可以对此文件拥有的permission

　　group：相当于Linux里group的permission

　　namedgroup：定义了额外的组可以对此文件拥有的permission

　　mask：定义了nameduser,namedgroup和group的最大权限

　　other：相当于Linux里other的permission
	
##机制使用方法  

* "ACL的基本命令"

　　ACL的主要命令有2个：getfacl和setfacl。Getfacl用于或取文件或者目录的ACL权限信息，而setfacl则用于设置文件或者目录的ACL权限信息。下面分别对他们的使用进行简单介绍。
　　Getfacl-获取ACL权限信息
　　getfacl命令用于获取文件的ACL权限信息，其基本用法为：getfacl[文件/目录名]。
　　值得注意的是：即使该文件系统上没有开启ACL选项，getfacl命令仍然可用，不过只显示默认的文件访问权限，即与ls-l显示的内容相似。
    Setfacl-设置ACL权限
    为了设置文件的ACL权限，需要使用setfacl命令来详细设置文件的访问权限，其基本用法如下：
    setfacl–[参数][文件/目录]，其常用的参数及作用如下所示：
    -m：建立一个ACL规则
    -x：删除一个ACL规则
　　-b：删除全部的ACL规则
　　-set：覆盖ACL规则
　　下面来详细介绍如何使用setfacl来设置文件/目录的ACL权限。

　　(1)添加/修改ACL规则

　　需要使用-m选项来进行操作。

　　举个例子，使用该命令为用户gavin和组test设置acl_test文件的读写权限，并使用getfacl查看设置结果：

　　$setfacl-mu:gavin:rw,g:test:racl_test

　　$getfaclacl_test

　　#file:acl_test

　　#owner:gavin

　　#group:gavin

　　user::rw-

　　user:gavin:rw-

　　group::rw-

　　group:test:r--

　　mask::rw-

　　other::r—

　　在上面的命令示例中，可以清楚地看到加粗部分user:gavin、group:test、mask这3个ACLEntry的出现，表明对文件进行了ACL权限设置，否则，不会出现该标识。为了进一步验证，我们使用ls-l来查看该文件的权限位中是否多了“+”这个标识位，如下所示：

　　$ls-lacl_test

　　-rw-rw-r--+1gavingavin012-2011:49acl_test

　　其中，user:gavin、group:test为我们设置的访问权限，而mask::rw为自动添加的内容。

　　(2)删除ACL规则

　　使用-x选项可以方便地删除指定用户对指定文件/目录的访问权限。

　　以下示例删除用户gavin对文件acl_test的访问权限：

　　$setfacl-xu:gavinacl_test

　　$getfaclacl_test

　　#file:acl_test

　　#owner:gavin

　　#group:gavin

　　user::rw-

　　group::rw-

　　group:test:r--

　　mask::rw-

　　other::r--

　　$ls-lacl_test

　　-rw-rw-r--+1gavingavin012-2011:49acl_test

　　通过上述2段ACL权限显示的对比可以清楚地看到：用户gavin对于文件acl_test的访问权限已经完全删除了，表现为user:gavin:rw-已经不存在了。这里提醒注意的是：我们不能够通过setfacl命令来指定删除用户/组对文件/目录的某一个特定权限(如r、w或者x)。同时，也可以看到，使用ls-l命令显示文件的9个权限位还是没有改变，因为改变的只是ACL权限，而不是最基本的user、group和others权限。

　　(3)删除文件/目录的所有ACL规则

　　使用-b选项可以删除文件/目录的ACL权限。如下命令将删除文件acl_test的所有ACL权限。可以看到，使用getfacl命令来查看是，mask项已经消失，即该文件已经没有了所有的ACL权限：

　　$setfacl-bacl_test

　　$getfaclacl_test

　　#file:acl_test

　　#owner:gavin

　　#group:gavin

　　user::rw-

　　group::rw-

　　other::r—

　　(4)覆盖文件的原有ACL规则

　　需要使用--set选项。此处需要强调一下-m选项和--set选项的区别：-m选项只是修改已有的配置或是新增加一些;而--set选项和-m不同，它会把原有的ACL项全都删除，并用新的替代。另外，--set选项的参数中一定要包含UGO的设置，不能象-m一样只是添加ACL就可以了。

　　以下示例该选项的使用方法：

　　$setfacl--setu::rw,g::rw,o::r,u:gavin:rwx,g:test:rxacl_test

　　$getfaclacl_test

　　#file:acl_test

　　#owner:gavin

　　#group:gavin

　　user::rw-

　　user:gavin:rwx

　　group::rw-

　　group:test:r-x

　　mask::rwx

　　other::r--

　　$ls-lacl_test

　　-rw-rwxr--+1gavingavin012-2011:49acl_test

　　这里需要提醒注意的是：上述acl_test文件的权限标识位中的group的rwx权限，并不是表明acl_test文件所属用户的用户组对其有x权限，实际上只具有rw权限，而是因为在group:test:r-x中指定了test这个组具有x权限，所以ACL机制在这个标识位上进行了体现，在实际的应用中要特别注意，切记不要弄混淆了。

　　(5)其他选项

　　除了上述介绍的4类用法外，setfacl还可以使用如下一些选项，如下表，供大家在实际使用中参考：

　　为目录创建默认ACL

　　在日常的使用过程中，经常是通过对目录来设定ACL权限来满足应用的需求，而很少仅仅通过设置特定的文件来实现，因为这样做比较繁琐和低效。因此，下面就介绍如何来为目录创建默认的ACL。

　　如果希望在一个目录中新建的文件和目录都使用同一个预定的ACL，那么我们可以使用默认ACL(DefaultACL)。在对一个目录设置了默认的ACL以后，每个在目录中创建的文件都会自动继承目录的默认ACL作为自己的ACL。

　　具体的设置命令为：setfacl-d[目录名]。

　　下面的例子对新建的test目录进行ACL权限设置，并在其中新建了test1和test2文件，来看看默认ACL的设置情况。

　　首先，对文件夹test进行ACL权限查看如下：

　　$getfacltest

　　#file:test

　　#owner:gavin

　　#group:gavin

　　user::rwx

　　group::rwx

　　other::r-x

　　接着，对其进行权限设置如下：

　　$setfacl-d-mg:test:rtest

　　$getfacltest

　　#file:test

　　#owner:gavin

　　#group:gavin

　　user::rwx

　　group::rwx

　　other::r-x

　　default:user::rwx

　　default:group::rwx

　　default:group:test:r--

　　default:mask::rwx

　　default:other::r-x

　　可以看到，经过setfacl设置后，该文件夹的权限增加了以default开始的几项，表明设置成功。

　　然后，建立两个新的文件，然后查看他们的ACL权限信息如下：

　　$touchtest1test2

　　$getfacltest1

　　#file:test1

　　#owner:gavin

　　#group:gavin

　　user::rw-

　　group::rwx#effective:rw-

　　group:test:r--

　　mask::rw-

　　other::r--

　　$getfacltest2

　　#file:test2

　　#owner:gavin

　　#group:gavin

　　user::rw-

　　group::rwx#effective:rw-

　　group:test:r--

　　mask::rw-

　　other::r--

　　可以清楚地看到：文件test1和test2自动继承了test设置的ACL。

　　备份和恢复ACL

　　目前，Linux系统中的主要的文件操作命令，如cp、mv、ls等都支持ACL。因此，在基本的文件/目录操作中可以很好地保持文件/目录的ACL权限。然而，有一些其它的命令，比如tar(文件归档)等常见的备份工具是不会保留目录和文件的ACL信息的。因此，如果希望备份和恢复带有ACL的文件和目录，那么可以先把ACL备份到一个文件里，待操作完成以后，则可以使用--restore选项来恢复这个文件/目录中保存的ACL信息。
##机制的安全分析



##ACL缺陷分析与改进思路
  
	与原有的DAC功能相比，ACL虽然实现了更细粒度的访问控制，但它还存在一些不足和需要改进的地方。
	1. 现在的ACL机制中，访问类型仍为读,写和执行三种，粒度还是显得过粗，应该在访问控制类型这一方面进一步细化。
	1. 在Linux传统的访问控制机制中，有一个授权的概念，如果当前进程得到了授权，那么它就成为了授权进程了，就可一凌驾于文件系统的访问权限控制机制之上，不必做权限检查就可以进行操作。授权的目的是为了给一些特殊用户提供特权，但是这种授权方式存在了一些问题，因为授权进程总是被赋予了许多不必要的全能，没有达到最小权限的安全目标。ACL机制还是保留来授权的概念，它在检查进程是否授权之前做了必要的权限检查，但还是没有消除授权所带来的权限未最小化的问题。
	1. ACL在网络规模小的环境中是一种方便有效的访问控制方式，但是在需求复杂，网络规模较大的网中，ACL并不是一种很好的访问控制方式，因为ACL要对每个资源指定可以访问的用户或组以及相应的访问权限，如果网络中的资源很多，系统中会存储大量ACL表项，管理会变得很繁琐，而且当用户打开文件时，这些表象就要被读进内存，占用一定的内存空间。
	1. 将时间控制引入到ACL机制中。如果某个用户登录以后长时间没有使用，那么取消他的访问权限，如果想要访问的话重新登录，这样可以避免没有访问权限的用户使用此系统。
##总结
    
