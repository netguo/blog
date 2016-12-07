
###一. 先安装brew环境，按照官网。  

  
```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"   
```    
安装后检测一下，是否安装成功。  
```bash
brew doctor  
```      
如果不成功则根据提示进行相应的解决。  
   
###二. 使用 RVM 安装 Ruby  
```
curl -L https://get.rvm.io | bash -s stable --ruby  
```
为避免出现问题，可执行以下命令安装 Ruby 2.0.0:  

```  
rvm install 2.0.0   
rvm rubygems latest  
```
可以执行 `ruby --version` 命令来查看现在使用的 Ruby 版本，确保正在使用的是 Ruby 2.0.0  
  
###三.本地安装octopress  
```
git clone git://github.com/imathis/octopress.git octopress
cd octopress
```    
如果有自己的独立域名：`echo 'your-domain.com' >> source/CNAME`  
并在自己的域名中设置：顶级域名设置，192.30.252.153和192.30.252.154解析。 非顶级域名可以直接指定github项目地址。
  
###四.安装 Octopress 所必需的依赖项  
```
gem install bundler
```  
安装bunder时，如果报以下的错误  :  

```
ERROR:  Could not find a valid gem 'bundler' (>= 0), here is why:
      Unable to download data from https://rubygems.org/ - Errno::ETIMEDOUT: Operation timed out - connect(2) (https://rubygems.org/latest_specs.4.8.gz)
ERROR:  Possible alternatives: bundler     
```  
是国内的GFW的原因，可以不通过rubygems.org安装，网上找到一个淘宝的镜像，根据淘宝镜像地址进行安装。如下：    

```  
$ gem sources --remove https://rubygems.org/
$ gem sources -a http://ruby.taobao.org/
$ gem sources -l
*** CURRENT SOURCES ***
http://ruby.taobao.org
请确保只有 ruby.taobao.org
$ gem install bundler  
```  
继续安装bundler，`bundle install`，安装时可能会报以下的错误：    

```  
An error occurred while installing celluloid (0.16.0), and Bundler cannot continue.
Make sure that `gem install celluloid -v '0.16.0'` succeeds before bundling.  
```  
按照提示，`gem install celluloid -v '0.16.0'`，安装所缺的依赖项即可。    

###五. 运行octopress，并发布文章    
安装默认主题：`rake install`  
本地预览：`rake preview`，默认使用4000端口，输入localhost:4000，既可本地预览。  
部署到github：`rake setup_github_pages`，运行命令后，会提示输入远程地址，输入github项目地址即可。  
发布到github:     
```
rake generate  
rake deploy  
```  
把资源提交到source分支。  
  
```
git checkout source  
git add .  
git commit -m "修改..."  
git push origin source  
```  
发布新的文章：`rake new_post "Post Title"`，文章的md的文件会自动生成，在目录octopress/source/_posts下。  


###六. 个性化修改   
     
####新主题安装：  
Octopress 本身有很多第三方主题可以直接安装使用。[octopress第三方主题](https://github.com/imathis/octopress/wiki/3rd-Party-Octopress-Themes)  
选择一个主题，并安装该主题github上的操作进行操作即可。
例如Bootstrap theme:    

```  
$ cd octopress
$ git submodule add GIT_URL .themes/THEME_NAME
$ rake install['THEME_NAME']
$ rake generate 
```  
如果`rake install['THEME_NAME']`报错：'no matches found'，则：`rake install\['THEME_NAME'\]`。  

####_config.yml配置：  
  
 _config.yml主要配置博客作者，时间显示方式等基本的信息。    
 
   
```
 # ------     Main Configs    -----   #  
 url: http://netguo.cn blog的url  
 title: My Blog -- Netguo  ＃博客上面的标题  
 subtitle: stay hungry stay foolish  ＃副标题  
 author: net guo  ＃作者  
 simple_search: https://www.google.com/search  ＃搜索框的默认搜索  
 description:net guo的个人博客小站  ＃博客的描述
 date_format: "%Y 年%-m 月%-d 日" ＃时间格式，这样配置会进行相应的汉化
 ＃ －－RSS / Email (optional) 的描述：
 subscribe_rss: /atom.xml
 subscribe_email:
 email: netguo@gmail.com   #－－RSS 将会显示该邮件地址 
 
 # -----    Jekyll & Plugins    ----- #  
 root: / ＃默认根目录
 permalink: /blog/:title/  ＃生成文章的目录推荐此格式，默认会带有日期前缀
 source: source   ＃source目录
 destination: public
 plugins: plugins
 code_dir: downloads/code
 category_dir: blog/categories
 category_title_prefix: "分类：" # 修改分类前缀
 markdown: kramdown  ＃markdown解析方式，推荐kramdown
 paginate: 10          # 每页的博客数目，如果超过该数目则分页
 paginate_path: "posts/:num"  # Directory base for pagination URLs eg. /posts/2/
 recent_posts: 5       # 右侧显示过去博客的数目
 excerpt_link: "阅读全文 &rarr;"  #  阅读全文
 excerpt_separator: "<!--more-->"   ＃文章中含有该标示，在文章列表中，后面的内容将不会显示，以阅读全文代替。 
 
 # -----  3rd Party Settings   ------ #  
 # Disqus Comments
 disqus_short_name:
 disqus_show_comment_count: false
```

####Rakefile配置    
保持默认设置即可，如果想更改首页目录，更改`blog_index_dir`即可。
  
```  
public_dir      = "public"    # compiled site directory
source_dir      = "source"    # source file directory
blog_index_dir  = 'source'    # 主目录
deploy_dir      = "_deploy"   # deploy directory (for Github pages deployment)
stash_dir       = "_stash"    # directory to stash posts for speedy generation
posts_dir       = "_posts"    # directory for blog files
themes_dir      = ".themes"   # directory for blog files
new_post_ext    = "md"  # default new post file extension when using the new_post task
new_page_ext    = "md"  # default new page file extension when using the new_page task
server_port     = "4000"      # port for preview server eg. localhost:4000  
```

####自定义主题模板：    
修改主页内容 :  
默认source下的index.html。  
  
修改每个博客的内容：  
每个页面的内容为_layout下的post，post包含_include下的article.html。   
 
修改主页导航栏，侧边栏，底部footer内容：  
/source/_includes/custom 这个目录下的相关文件来修改。  

插件安装：安装第三方插件以实现相应效果，比如侧边栏显示 Twitter 时间线等等。 主要与 /plugins 这个目录有关。  

样式修改：字体，配色等通过样式表修改的属性。主要通过修改 /sass/custom/_styles.scss 来实现。 
  
####生成文章目录。  
在文章中添加如下代码即可：  
  
```  
* list element with functor item
{:toc}  
```

参考:  
[octopress官方文档](http://octopress.org/docs/)    
[如何安装homebrew](http://brew.sh/index_zh-cn.html)  
[如何解决国内安装gem问题](http://github.kimziv.com/2013/07/19/how-to-install-ruby-gems-in-china/)  


 
 
 
 
 

