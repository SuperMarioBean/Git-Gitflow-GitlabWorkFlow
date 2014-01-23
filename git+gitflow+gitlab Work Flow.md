#git+gitflow+gitlab演示中出现的权限问题
##情境重现

+ manager设置了develop和master分支为protected状态后
+ developer使用以下git命令推送本地developer个人分支推送到受protected的分支

		git push origin fuBin_ios7_fix:master

+ 服务器并未对这种超越权限行为组织，导致推送成功
		
##问题原因

经分析，原因并非是gitlab权限无效。昨晚的bug主要是因为我在同一台机器上同事扮演两个角色而造成的。

+ git实际上是通过本机上面的config文件中user设定来判定，究竟是谁将代码推送到服务器上的。你可以使用一下git命令输出user的用户名和邮箱地址

		git config user.name
		git config user.email
		
	但是实际上，git通过两套config文件来保证，一个全局config文件和一个本地版本库的config文件，这两个文件同时作用决定你推送时所使用的账号，规则是：
	
		priority: local > global
	
	你可以通过以下git命令输出user的全局用户名和邮箱地址设定
	
		git config --global user.name
		git config --global user.email
		
	全局和本地的config文件的位置你可以通过以下的路径查看：
	
		{{HOME}}/.gitconfig 						#global
		{{PATH_TO_YOUR_REPOSITORY}}/.git/CONFIG		#local
		
+ 昨天的错误在于，虽然我考虑了用户切换的问题，但是我的切换脚本只设置了全局用户名和邮箱，而并没有修改版本库中的用户名和邮箱设置。所以导致了developer版本库中用户设置没有改变。

	>实际上这个错误可能出现的更早，就是说我在使用同一台机器拉取manager的代码时，本身使用的就是带有manager用户名和邮箱设置的git，这就导致了这个developer的本地库的本地设置默认成了manager的用户设置，而修改全局用户设置的脚本是会被覆盖的，这样就会被服务器认为你仍旧是manager，从而无视对于developer角色的权限设置，直接push
	
+ 实际上这个错误在正常情况下很难发生，因为我在一台机器上复现两个权限不同的角色。但是如果你在一台机器上使用多个用户账号来进行推送，这个问题也有可能发生，比如你有github、bitbucket、gitlab多个代码推官网站的账号，而你又希望他们通过不同的email联系你，这就导致了你有多个不同的账号设置，虽然代码库在拉取到本地后用户会自动设置成为你的服务器版本的用户，但是我仍旧建议你在这种情况检查本地设置文件中关于用户的设置。

##继续昨晚的话题

由于昨晚没有掩饰完毕，所以我只好出一份使用文档。大致的developer和manager之间的使用流程如下。
###1 manager
+ 配置 
	
	manager创建好项目，通过gitflow设置好develop和master分支，设定protected，加入新的成员，并配置好相应的访问权限后，即可开始工作

###2 developer
+ 工作

	developer通过下面的git命令拉取版本库到本地

		git clone {{GIT_URL_TO_YOUR_SERVER_REPOSITORY}}
	
	通过gitflow命令创建本地分支，所有的工作都应该在独立的分支上进行
	
		git flow feature start {{YOUR_NAME}}_{{FEATURE_NAME}}
		
	工作，提交代码和日志
	
		git add {{FILE_YOU_WANT_TO_STAGE}}
		git commit -m "{{COMMIT_LOG}}"	
	
	经过多次提交，本地测试后变基调整之后，通过以下git命令将本地自己的分支push到服务器对应的分支上（实际上是在服务器上新创建的分支）
	
		git push origin {{SAVE_AS_YOUR_BRANCH_NAME}}
	
	然后通过gitlab发起merge request，填写你希望合并入的分支等信息，确认。

###2 manager	
+ 合并

	manager收到merge request的请求后，通过以下命令将远程分支的引用获取到本地，即当前developer新创建的分支
	
		git fetch
		
	>此时不是以本地分支存在，而是保存在 .git/refs/heads 之外（.git/refs/remotes/origin/之中）的远程分支。你通过
	
	>		git branch
	
	>是无法看到这个fetch到得分支，只有通过
	
	>		git branch -a
	
	>才能看到，并且在分支名称前面带有一个origin/
	
	查看新分支于此前的版本之间的commit级别的差异
	
		git log master..origin/{{NEW_BRANCH_NAME}}
	
	确认无误后可以使用下面的命令进行合并
	
	>合并可能需要处理冲突，对于冲突的理解后面再更新
	
		git merge --no-ff origin/{{NEW_BRANCH_NAME}}
		
	>推荐使用--no-ff参数，这样会针对合并新创建一个commit，可以保持分之间的条理清晰，便于阅读
	
	合并后一般manager应该进行本地测试，如果有误，可以通过以下命令返回合并前的状态
	
		git reset --hard HEAD^
		或	git reset --hard develop@{1}
	
	反复检查无误，即可推送
	
		git push
		
	合并推送后在gitlab的分支树上就会有合并体现，此后再去关闭此前developer请求的merge request。

###3 developer
+ 确认

	当manager的merge request关闭后，developer会得到确认信息，此时执行
	
		git fetch
		
	和此前一样，你将获得所有分支的引用，包括合并后跟新的develop分支。处于当前分支
	
		git checkout develop
		
	合并
	>实际上git pull等效于git fetch + git merge,将其分开只是为了更安全，因为在merge之前，你可能想不到你fetch到的是什么样的引用
	
		git merge origin/develop
		
	而此时developer本地的分支已经无用，所有删除之
	
		git branch -d {{YOUR_BRANCH_NAME}}
	
	至此，一个简单的工作流程完成