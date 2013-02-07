---
layout: post
title: "Chocolatey, 我爱你 （apt-get for Windows）"
date: 2013-02-07 14:28
comments: false
categories: 
 - chocolatey
 - powershell
 - windows automation
 - devops
---

巧克力在很多文化里都代表了美好的事物。如果你是Windows的使用者，可能Windows复杂、丑陋的包管理正在把你折腾的很惨。*NIX平台上的各种优雅的包管理工具这时让人非常眼馋。

1. 安装
Chocolatey的安装过程简单至极，在http://chocolatey.org/上最醒目的地方有一行命令，把它复制到命令行中运行，只要一分钟左右，安装就完成了。在命令下键入

chocolatey help

可以验证安装是否成功。

2. 初体验
默认情况下，Chocolatey会把自己安装到C:\Chocolatey目录下。该目录下还有三个子目录，它们的作用分别是

bin - Chocolatey自身的命令，以及通过Chocolatey安装的某些软件会在bin下增加一个*.bat的快捷方法。
chocolateyInstall - Chocolatey运行时的程序以及各种log。
lib - 安装过程中下载的包。

是时候尝试一下Chocolatey的威力了，在命令行上键入

chocolatey install 7zip

此时会出现下载的进度信息。耐心等一会儿，Chocolatey会告诉你安装完成。

“Finished installing '7zip' and dependencies - if errors not shown in console, none detected. Check log for errors if unsure”

这之后检查你的Program Files(或者Program Files (x86))下应该包含7-Zip目录了。

3. 安装自己的包

在我写这篇博客的时候，我的机器上并没有中文输入法。在Chocolatey上寻找未果后（http://chocolatey.org/packages?q=google+pinyin），我只能自己为Google Pinyin创建一个Chocolatey的安装包。研究过C：\Chocolatey\lib\下面的7Zip的包文件后，我们可以获知创建Chocolatey包的方法。

Chocoletay包一个满足特定目录结构的NuGet包。NuGet（http://nuget.org/）是另一个Windows上令人心动的工具，它是一种特定的包格式，类似于.deb .rpm，同时它也具有版本化的包管理功能。与Chocolatey的不同在于，NuGet关注在开发人员使用的包，而Chocoletay更关注最终用户可用的软件包。既然Chocolatey包就是NuGet包，可以用命令

NuGet.exe pack ****.nuspec

创建。如果你没有安装NuGet没有关系，Chocolatey包含了一个NuGet的二进制执行文件，可以使用命令

cpack.bat pack ****.nuspec

创建。cpack.bat可在C:\Chocolatey\bin目录下找到。

不过Chocolatey的包还需要一些特殊的内容，才能被Chocolatey识别并安装。查看7Zip的安装文件后，可以看到我们需要在包的顶级目录下创建一个tools目录，并在下面包含一个chocolateyInstall.ps1文件，该文件是安装包的（PowerShell）代码。 综上所述，我们需要很少的步骤，就能创建Google Pinyin的包：

--- C:/GooglePinyinPackage/google-pinyin.nuspec ---
<?xml version="1.0"?>
<package>
    <metadata>
        <id>google-pinyin</id>
        <version>1.0</version>
        <title>Google Pinyin</title>
        <authors>Han Kai</authors>
        <owners>Han Kai</owners>
        <requireLicenseAcceptance>false</requireLicenseAcceptance>
        <description>Google pinyin installation for Chinese requirement.</description>
        <copyright>Copyright 2012</copyright>
        <tags>google pinyin</tags>
    </metadata>
    <files>
        <file src="chocolateyInstall.ps1" target="tools/chocolateyInstall.ps1" /> 
    </files>
</package>

--- C:/GooglePinyinPackage/chocolateyInstall.ps1 ---
try {
    Install-ChocolateyPackage 'google-pinyin' 'exe' '/s' 'http://dl.google.com/pinyin/v2/GooglePinyinInstaller.exe'
} catch { 
    Write-ChocolateyFailure 'reflector' "$($_.Exception.Message)"
}

此处使用了Chocolatey提供的Install-ChocolateyPackage函数，它需要四个参数，id（google-pinyin，这个要和google-pinyin.nuspec中的一致），安装文件类型，安装参数（/s是silent installation）和安装文件的下载路径。


运行
cpack google-pinyin.nuspec

cinst googe-pinyin -source C:\GooglePinyinPackage


























