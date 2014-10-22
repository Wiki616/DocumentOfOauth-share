DocumentOfOauth-share
=====================
1.所有下面要求的，就是建立在搭好了flask框架环境的基础上，flask环境搭建见flask_setup.txt，另外由于只是举个例子，所以不相关的文件都删了
flask一个项目的目录结构主要如下：
app
--static
-----pic//图片素材
-----js//js文件
-----font//字体文件
--templates
--views.py

2.Oauth部分：
1）代码构成：除了google因为js中发送ajax请求可以被google服务器允许，所以google的oauth部分全部写在google.js中，其余oauth部分一般分成js文件盒views中的一个函数部分来处理
2）各网站oauth申请地址：
	google：https://console.developers.google.com/
	github：https://github.com/settings/applications
	QQ：http://connect.qq.com/manage/login
	人人：http://wiki.dev.renren.com/wiki/Authentication
3）具体html文件 js文件 以及views.py中的函数之间的关系
html中对应的button名为googlelogin，qqlogin这种命名方式Xlogin，在对应x.js文件中，当id名为xlogin触发click事件后执行js文件中指令
一般js中代码结构（qq举例）如下
$(function() {
	$('#qqlogin').click(function() {
		login();
	});

	var clientId = "client_id=101157515";	//client_id , 申请后获得
	var redirect_uri = "redirect_uri=http://osslab.msopentech.cn/qq"; //回调地址，跟网上填写对应
	var scope = "scope=get_user_info";	//需要调用api的类型，各个网站不同，这里指仅获得用户信息
	var state="state=osslab";	//有的需要填写下
	var response_type = "response_type=code" //返回类型，这个具体看网上文档
 	
	function login() {
		var arr = [clientId,redirect_uri,scope,state,response_type];
		var url = "https://graph.qq.com/oauth2.0/authorize?" + arr.join("&");
		location.href = url; //跳转到指定的url
	}
});

views.py 部分
@app.route('/qq')
def qq():
	code = request.args.get('code')	//根据具体情况不同而不同，得到url中某个特定的参数code
	#print code
	url = '/oauth2.0/token?grant_type=authorization_code&client_id=101157515&client_secret=018293bdbc15ddfc84306234aa34aa6c&redirect_uri=http://osslab.msopentech.cn/qq&code=' + code + '&state=osslab' //生成新的url
	httpres = query_info('graph.qq.com',url,2) //query_info函数在views.py中，第一个参数为网址根目录，url为后接url（包含参数），1是http2是https
	url_ori = httpres.read()
	start = url_ori.index('=')
	end = url_ori.index('&')
	access_token = url_ori[start+1:end]	//字符串处理得到access_token
	//通过access_token获取info
	url = '/oauth2.0/me?access_token=' + access_token	//生成新的url参数列表
	httpres = query_info('graph.qq.com',url,2)	//和上述过程类似，具体情况参考文档
	info = httpres.read()// 获取信息,有些信息不需要处理可以直接转成json文件
	info = info[10:-4] //处理，去掉冗余元素，得到可以转成info的文件
	info = json.loads(info)	//转成info文件
	openid=info['openid'] //取出特定的值
	appid = info['client_id']
	url = '/user/get_user_info?access_token=' + access_token + '&oauth_consumer_key=' +appid +'&openid=' + openid
	httpres = query_info('graph.qq.com',url,2)
	info = json.loads(httpres.read())	//得到用户信息,QQ比其他有些多了一部，有些只要两次https请求
	return render_template("qq.html",name=info['nickname'],pic=info['figureurl'])//在模板中嵌入某些信息
	
Google：
// google oauth 跟前面的类似，跳转到回调地址
$(function() {
	$('#googlelogin').click(function() {
		login();
	});

	var clientId = "client_id=304944766846-7jt8jbm39f1sj4kf4gtsqspsvtogdmem.apps.googleusercontent.com";
	var localurl = location.href;
	var redirect_uri = "redirect_uri=" + localurl + "google";
	var scope = "scope=https://www.googleapis.com/auth/userinfo.profile+https://www.googleapis.com/auth/userinfo.email";
	var response_type = "response_type=token";
	
	function login() {
		var arr = [clientId,redirect_uri,scope,response_type];
		var url = "https://accounts.google.com/o/oauth2/auth?" + arr.join("&");
		location.href = url;
	}
});


 $(function() {
                        $(document).ready(function() {
                                Display();
                        });

                        function getUrlparam(name) {
                                var reg = new RegExp("(^|&)"+ name +"=([^&]*)(&|$)");
                                var r = window.location.search.substr(1).match(reg);
                                if (r!=null) return unescape(r[2]); return null;
                        }

                        function Display() {
                                var name = "access_token";
                                var url = window.location.href;
                                var theRequest = new Object();
                                if (url.indexOf("#")!=-1) {//google返回url跟其他不同，中间是用#隔开的，不是？
                                        var str = url.substr(url.indexOf("#")+1);
                                        var strs = str.split("&");
                                        for (var i = 0;i < strs.length;i ++)
                                        {
                                        theRequest[strs[i].split("=")[0]] = unescape(strs[i].split("=")[1]);
                                        }
										//上面是获取name为access_token的信息
                                        newurl = "https://www.googleapis.com/oauth2/v1/userinfo?access_token=" + theRequest[name];
                                        $.ajax({	//发送ajax请求
                                                type : "GET",
                                                url : newurl,
                                                dataType : "jsonp",
                                                success: function(msg) {
                                                         $("#name").html('<p style = "color:white">' + msg.name  + "</p>");
                                                         $("#photo").html('<img src = "' + msg.picture + '" height = "30" width = "30"/>');
							 document.cookie = 'username=' + msg.name;
							 document.cookie = 'picurl=' + msg.picture;
							 var pre = window.location.host;
                                                         for (var i = 1;i <= 9;i++) {
                                                                name = "#image" + i.toString();
                                                                url = "course?type=" + i.toString();
                                                                $(name).attr("href",url);
                                                         }
                                                },
                                                error : function(msg) {
                                                        alert("something is wrong!\n" + msg.d);
                                                }
                                        });
                                }
                        }
                });


				
3.分享功能
<a title="转发至QQ空间" charset="400-03-8" id="s_qq" href="http://sns.qzone.qq.com/cgi-bin/qzshare/cgi_qzshare_onekey?url=你的网站地址" target="_blank"><img src="http://static.youku.com/v1.0.0691/v/img/ico_Qzone.gif" /></a>
<a title="转发至人人网" charset="400-03-7" id="s_renren" href=http://share.renren.com/share/buttonshare.do?link=你的网站地址&title=标题 target="_blank"><img src="http://static.youku.com/v1.0.0691/v/img/ico_renren.gif" /></a>
<a title="转发至新浪微博" charset="400-03-10" id="s_sina" href="http://v.t.sina.com.cn/share/share.php?appkey=2684493555&url=你的网站地址&title=Uid=&source=&sourceUrl=" target="_blank"><img src="http://static.youku.com/v1.0.0691/v/img/ico_sina.gif" /></a>
<a title="分享到腾讯朋友" charset="400-03-19" id="s_pengyou" href=http://sns.qzone.qq.com/cgi-bin/qzshare/cgi_qzshare_onekey?to=pengyou&url=你的网站地址&title=%标题 target="_blank"><img src="http://static.youku.com/v1.0.0691/v/img/ico_pengyou.png" /></a>
<a title="推荐到豆瓣" charset="400-03-17" id="s_douban" href=http://www.douban.com/recommend/?url=你的网站地址&title=标题  target="_blank"><img src="http://static.youku.com/v1.0.0691/v/img/ico_dou_16x16.png" /></a>
其实就是一个图片链接，链接到门户网站指定的地址即可，可以使用js来生成正确的url值，赋给a标签的href值