
# 1 下载数据
用诺禾的软件下载 ./lnd ogin -u 用户名 -p 密码登录
/lnd cp oss:// 目录/文件 本地目录下载，直接下载目录就ok，软件会统计文件完整度.

# 2 数据质控
！！！不要直接拿公司的clean data来做，我做到peak注释了才发现比对率很差，浪费了很多时间，还是自己质控好一点。
公司的raw——data有两个G质控完剩下900M，当时应该想到不对的。
还是用trim_galore软件

```
cat config.raw  |while read id;
do echo $id
arr=($id)
fq2=${arr[2]}
fq1=${arr[1]}
sample=${arr[0]}
nohup  trim_galore -q 25 --phred33 --length 35 -e 0.1 --stringency 4 --paired -o  ../clean/  $fq1   $fq2  &   ##注意这里最好加上--length 15 因为
done                                                                                                       ##如果保留了空reads，软件不能识别+号 
```
