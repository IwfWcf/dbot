# coding=utf-8
from bottle import *
import hashlib
import xml.etree.ElementTree as ET
import urllib2
import json

# 基于 Kingson 的「豆瓣电影查询助手 3.0」代码进行修改，优化摘要模式，电影链接使用移动版链接。增加了图书查询以及结果列表模式。本来还想做评论查询的，但因为需要豆瓣的高级 API 权限而无法实现。 
# 未解决问题：
# 摘要中的海报/封面被自动截取，无法完整显示

app = Bottle()
 
@app.get('/weixin')
def checkSignature():
    # 这里是用来做接口验证的，从微信 Server 请求的 URL 中拿到“signature”,“timestamp”,"nonce"和“echostr”，
    # 然后再将 token, timestamp, nonce 三个排序并进行 Sha1 计算，并将计算结果和拿到的 signature 进行比较，
    # 如果相等，就说明验证通过。
    # 附微信 Server 请求的 Url 示例：http://yoursaeappid.sinaapp.com/?signature=730e3111ed7303fef52513c8733b431a0f933c7c&echostr=5853059253416844429&timestamp=1362713741&nonce=1362771581
	
    token = "doubanbot"  # 在微信公众平台上设置的 TOKEN
    signature = request.GET.get('signature', None)
    timestamp = request.GET.get('timestamp', None)
    nonce = request.GET.get('nonce', None)
    echostr = request.GET.get('echostr', None)
    tmpList = [token, timestamp, nonce]
    tmpList.sort()
    tmpstr = "%s%s%s" % tuple(tmpList)
    hashstr = hashlib.sha1(tmpstr).hexdigest()
    if hashstr == signature:
        return echostr
    else:
        return None
 
 
def parse_msg():
    # 这里是用来解析微信 Server Post 过来的 XML 数据的，取出各字段对应的值，以备后面的代码调用，也可用 lxml 等模块。
    
    recvmsg = request.body.read()  
    root = ET.fromstring(recvmsg)
    msg = {}
    for child in root:
        msg[child.tag] = child.text
    return msg
 
 
def query_info(qtype, keyword):
    # 这里使用豆瓣的电影/图书 search API，通过关键字查询电影/图书信息，这里的关键点是如果关键字中存在汉字，就需要先转码，才能进行请求
	
    if qtype == 0:
        urlbase = "https://api.douban.com/v2/movie/search"
    else:
        urlbase = "https://api.douban.com/v2/book/search"
    DOUBAN_APIKEY = "0206a0b355d01fb80613e21a01705f28"  # 这里需要填写你自己在豆瓣上申请的应用的 APIKEY
    searchkeys = urllib2.quote(keyword.encode("utf-8"))  # 如果 Content 中存在汉字，就需要先转码，才能进行请求
    url = '%s?q="%s"&apikey=%s' % (urlbase, searchkeys, DOUBAN_APIKEY)
    resp = urllib2.urlopen(url)
    result = json.loads(resp.read())
    return result
 
 
def query_details(qtype, ID):
    # 这里使用豆瓣的电影/图书 subject/book API，通过电影/图书 ID 来获取电影/图书的信息。
    
    if qtype == 0:
        urlbase = "https://api.douban.com/v2/movie/subject/"
    else:
        urlbase = "https://api.douban.com/v2/book/"
    DOUBAN_APIKEY = "0206a0b355d01fb80613e21a01705f28"  # 这里需要填写你自己在豆瓣上申请的应用的 APIKEY
    url = '%s%s?apikey=%s' % (urlbase, ID, DOUBAN_APIKEY)
    resp = urllib2.urlopen(url)
    item = json.loads(resp.read())
    return item
 
 
@app.post('/weixin')
def response_msg():
    # 这里是响应微信 Server 的请求，并返回数据的主函数
    # 基本思路：
    # 拿到 Post 过来的数据
    # 分析数据（拿到 FromUserName、ToUserName、CreateTime、MsgType 和 content）
    # 构造回复信息（将你组织好的 content 返回给用户）
    
    # 拿到并解析数据
    msg = parse_msg()
    # 设置返回数据模板
    # 纯文本格式
    welcome = """<xml>
             <ToUserName><![CDATA[%s]]></ToUserName>
             <FromUserName><![CDATA[%s]]></FromUserName>
             <CreateTime>%s</CreateTime>
             <MsgType><![CDATA[text]]></MsgType>
             <Content><![CDATA[%s]]></Content>
             <FuncFlag>0</FuncFlag>
             </xml>"""
    # 摘要条目
    detail = """<xml>
                <ToUserName><![CDATA[%s]]></ToUserName>
                <FromUserName><![CDATA[%s]]></FromUserName>
                <CreateTime>%s</CreateTime>
                <MsgType><![CDATA[news]]></MsgType>
                <ArticleCount>1</ArticleCount>
                <Articles>
                <item>
                <Title><![CDATA[%s]]></Title>
                <Description><![CDATA[%s]]></Description>
                <PicUrl><![CDATA[%s]]></PicUrl>
                <Url><![CDATA[%s]]></Url>
                </item>
                </Articles>
                <FuncFlag>1</FuncFlag>
                </xml> """
    # 搜索结果数量 == 2
    dlist = """<xml>
                <ToUserName><![CDATA[%s]]></ToUserName>
                <FromUserName><![CDATA[%s]]></FromUserName>
                <CreateTime>%s</CreateTime>
                <MsgType><![CDATA[news]]></MsgType>
                <ArticleCount>2</ArticleCount>
                <Articles>
                <item>
                <Title><![CDATA[%s]]></Title>
                <PicUrl><![CDATA[%s]]></PicUrl>
                <Url><![CDATA[%s]]></Url>
                </item>
                <item>
                <Title><![CDATA[%s]]></Title>
                <PicUrl><![CDATA[%s]]></PicUrl>
                <Url><![CDATA[%s]]></Url>
                </item>
                </Articles>
                <FuncFlag>1</FuncFlag>
                </xml> """
	# 搜索结果数量 >= 3
    tlist = """<xml>
                <ToUserName><![CDATA[%s]]></ToUserName>
                <FromUserName><![CDATA[%s]]></FromUserName>
                <CreateTime>%s</CreateTime>
                <MsgType><![CDATA[news]]></MsgType>
                <ArticleCount>3</ArticleCount>
                <Articles>
                <item>
                <Title><![CDATA[%s]]></Title>
                <PicUrl><![CDATA[%s]]></PicUrl>
                <Url><![CDATA[%s]]></Url>
                </item>
                <item>
                <Title><![CDATA[%s]]></Title>
                <PicUrl><![CDATA[%s]]></PicUrl>
                <Url><![CDATA[%s]]></Url>
                </item>
                <item>
                <Title><![CDATA[%s]]></Title>
                <PicUrl><![CDATA[%s]]></PicUrl>
                <Url><![CDATA[%s]]></Url>
                </item>
                </Articles>
                <FuncFlag>1</FuncFlag>
                </xml> """
    # 判断是否为新用户
    if msg["MsgType"] == "event" and msg["Event"] == "subscribe":
        echostr = welcome % (
            msg['FromUserName'], msg['ToUserName'], str(int(time.time())),
            u"欢迎使用 dbot，dbot 可以帮你查询豆瓣的电影和图书信息，输入 help 查看使用说明。")
        return echostr
    else:
        arr = msg["Content"].split()
        # 命令说明
        if len(arr) == 1 and arr[0] == "help":
            echostr = welcome % (
                msg['FromUserName'], msg['ToUserName'], str(int(time.time())),
                u"[movie 电影名] - 显示电影摘要\n[book 书名] - 显示书籍摘要\n[search movie 电影名] - 显示豆瓣电影的前 3 条搜索结果\n[search book 书名] - 显示豆瓣图书的前 3 条搜索结果")
            return echostr
        # 摘要模式
        if len(arr) == 2:
            if (arr[0] == "movie"):
                result = query_info(0, arr[1])
                item = query_details(0, result["subjects"][0]["id"])
                echostr = detail % (msg['FromUserName'], msg['ToUserName'], str(int(time.time())), item["title"], u"导演：" + item["directors"][0]["name"] + "\n" + u"豆瓣评分：" + str(item["rating"]["average"]) + "\n\n" + item["summary"], item["images"]["large"], item["mobile_url"])
                return echostr
            if (arr[0] == "book"):
                result = query_info(1, arr[1])
                item = query_details(1, result["books"][0]["id"])
                echostr = detail % (msg['FromUserName'], msg['ToUserName'], str(int(time.time())), item["title"], u"作者：" + item["author"][0] + "\n" + u"豆瓣评分：" + item["rating"]["average"] + "\n\n" + item["summary"], item["images"]["large"], item["alt"])
                return echostr
        # 搜索结果列表
        if len(arr) == 3 and arr[0] == "search":
            if (arr[1] == "movie"):
                result = query_info(0, arr[2])
                if result["total"] == 1:
                    item = query_details(0, result["subjects"][0]["id"])
                    echostr = detail % (msg['FromUserName'], msg['ToUserName'], str(int(time.time())), item["title"], u"导演：" + item["directors"][0]["name"] + "\n" + u"豆瓣评分：" + str(item["rating"]["average"]) + "\n\n" + item["summary"], item["images"]["large"], item["mobile_url"])
                    return echostr
                if result["total"] == 2:
                    echostr = dlist % (
                        msg['FromUserName'], msg['ToUserName'], str(int(time.time())),
					    result["subjects"][0]["title"] + "\n" + u"豆瓣评分：" + str(result["subjects"][0]["rating"]["average"]), result["subjects"][0]["images"]["large"], result["subjects"][0]["alt"] + "mobile",
					    result["subjects"][1]["title"] + "\n" + u"豆瓣评分：" + str(result["subjects"][1]["rating"]["average"]), result["subjects"][1]["images"]["medium"], result["subjects"][1]["alt"] + "mobile")
                    return echostr
                if result["total"] >= 3:
                    echostr = tlist % (
                        msg['FromUserName'], msg['ToUserName'], str(int(time.time())),
					    result["subjects"][0]["title"] + "\n" + u"豆瓣评分：" + str(result["subjects"][0]["rating"]["average"]), result["subjects"][0]["images"]["large"], result["subjects"][0]["alt"] + "mobile",
					    result["subjects"][1]["title"] + "\n" + u"豆瓣评分：" + str(result["subjects"][1]["rating"]["average"]), result["subjects"][1]["images"]["medium"], result["subjects"][1]["alt"] + "mobile",
					    result["subjects"][2]["title"] + "\n" + u"豆瓣评分：" + str(result["subjects"][2]["rating"]["average"]), result["subjects"][2]["images"]["medium"], result["subjects"][2]["alt"] + "mobile")
                    return echostr
            if (arr[1] == "book"):
                result = query_info(1, arr[2])
                if result["total"] == 1:
                    item = query_details(1, result["books"][0]["id"])
                    echostr = detail % (msg['FromUserName'], msg['ToUserName'], str(int(time.time())), item["title"], u"作者：" + item["author"][0] + "\n" + u"豆瓣评分：" + item["rating"]["average"] + "\n\n" + item["summary"], item["images"]["large"], item["alt"])
                    return echostr
                if result["total"] == 2:
                    echostr = dlist % (
                        msg['FromUserName'], msg['ToUserName'], str(int(time.time())),
					    result["books"][0]["title"] + "\n" + result["books"][0]["author"][0] + ", " + result["books"][0]["publisher"] + "\n" + u"豆瓣评分：" + result["books"][0]["rating"]["average"], result["books"][0]["images"]["large"], result["books"][0]["alt"],
					    result["books"][1]["title"] + "\n" + result["books"][1]["author"][0] + ", " + result["books"][1]["publisher"] + "\n" + u"豆瓣评分：" + result["books"][1]["rating"]["average"], result["books"][1]["images"]["medium"], result["books"][1]["alt"])
                    return echostr
                if result["total"] >= 3:
                    echostr = tlist % (
                        msg['FromUserName'], msg['ToUserName'], str(int(time.time())),
					    result["books"][0]["title"] + "\n" + result["books"][0]["author"][0] + ", " + result["books"][0]["publisher"] + "\n" + u"豆瓣评分：" + result["books"][0]["rating"]["average"], result["books"][0]["images"]["large"], result["books"][0]["alt"],
					    result["books"][1]["title"] + "\n" + result["books"][1]["author"][0] + ", " + result["books"][1]["publisher"] + "\n" + u"豆瓣评分：" + result["books"][1]["rating"]["average"], result["books"][1]["images"]["medium"], result["books"][1]["alt"],
					    result["books"][2]["title"] + "\n" + result["books"][2]["author"][0] + ", " + result["books"][2]["publisher"] + "\n" + u"豆瓣评分：" + result["books"][2]["rating"]["average"], result["books"][2]["images"]["medium"], result["books"][2]["alt"])
                    return echostr