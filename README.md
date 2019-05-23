#流程
解析proto文件生成swagger文件，配置jenkins，将更新的swagger文件，导入到yapi里面，从而自动生成yapi文档
###下载proto
#####更新 protobuf 源码
```go
go get github.com/protocolbuffers/protobuf/{proto,protoc-gen-go}
```
#####下载micro protobuf插件protoc-gen-micro
```go
go get github.com/micro/protoc-gen-micro
```
#####下载swagger protobuf插件protoc-gen-swagger
```sh
go get -u github.com/scholar-ink/protoc-gen-swagger
```
###下载google定义文件
下载http://cdn.udian.me/rpc/google/api.zip 并解压到/usr/local/Cellar/protobuf/3.5.1_1/include/google/api 目录下 （版本不同自行修改）

#开始使用

####1. 引入google api
```protobuf
import "google/api/annotations.proto";
```
####2.添加备注

```protobuf
syntax = "proto3";
option go_package = "common";  //设置go包名，与proto文件名一致
package retail.api.v1.common;
//商品-分类    --yapi 目录名称
service Common {
    //发送短信            --接口名称
    rpc SendMsg(SendMsgRequest) returns (SendMsgResponse) {
        option (google.api.http) = {
            post: "/v1/common/common/SendMsg" --接口地址
            body: "*"
        };
    }
}
message SendMsgRequest{
    string BusinessCode = 1;
    //手机号码.   --须添加到字段上方//开头.结尾
    string Mobile = 2;
    //短信类型1、注册验证码2、找回U点登录密码3、绑定手机号4、员工注册成功5、店铺创建成功6、商家注册成功.
    int32 Type = 3;
    //短信模板参数.
    repeated string Params = 4;
}

```
####3.执行protoc
```go
protoc --micro_out=. --go_out=. *.proto
protoc --micro_out=. --go_out=. --swagger_out=. *.proto
```

####4.添加dep，并dep ensure
```yaml
[[constraint]]
  name = "google.golang.org/genproto"
  source = "github.com/google/go-genproto"
```

####5.为yapi配置jenkins
```bash
cd $WORKSPACE
time=$(date -d "-2 min" +"%Y-%m-%d %H:%M:%S")
echo $time
checkModify(){
    std=$(git log --pretty=oneline --after="$time" $1)
    if [ -n "$std" ]; then
        return 1;
    else
        return 0;
    fi
}
for file in `ls -a proto`
do
    if [ x"$file" != x"." -a x"$file" != x".." ];then
        if [ -d "proto/$file" ];then
           `checkModify "proto/$file"`
           if [ $? -eq 1 ] ; then
            json=$(cat proto/$file/*.swagger.json | tr '\n' ' ')
            json=$(echo $json |sed -e s/[[:space:]]//g -e 's/"/\\"/g')
            var='{"type":"swagger","token":"xxxxxxx","merge":"mergin","json":"'$json'"}'
            curl -H "Content-type: application/json" -X POST -d $var http://test.yapi.udian.me/api/open/import_data
            echo "upload " proto/$file
           fi
        fi
    fi
done
```
####6.push生成的*.swagger.json文件,便会自动在yapi生成接口文档