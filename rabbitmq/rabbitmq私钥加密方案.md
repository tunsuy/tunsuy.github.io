# rabbitmq私钥加密方案


## 整体方案
1、kmc加密rabbitmq私钥密码a，存储到文件f1
2、rabbitmq启动前，kmc解密文件f得到明文密码b
3、通过安全随机数得到明文密码b的加密密钥c
4、采用 `rabbitmqctl encode 明文密码b 密钥c` 命令得到加密密码d
5、将密钥c写入临时文件f2
6、将密码d和临时文件f2写入rabbitmq.config配置文件中
7、启动rabbitmq
8、删除临时文件f2

## 私钥加密配置示例
```sh
{ssl_options, [{cacertfile,"/opt/rabbitmq/ca-cert.pem"},
			   {certfile,"/opt/rabbitmq/rabbitmq-cert.pem"},
			   {keyfile,"/opt/rabbitmq/rabbitmq-key.pem"},
			   {password,{encrypted,<<"encrypt_pwd">>}}
			   {versions, ['tlsv1.2']},
			   {ciphers,  [{dhe_rsa,aes_256_cbc,sha256},
						  {dhe_dss,aes_256_cbc,sha256},
						  {dhe_rsa,aes_128_cbc,sha256},
						  {dhe_dss,aes_128_cbc,sha256},
						  {dhe_dss,aes_256_cbc,sha},
						  {dhe_dss,aes_128_cbc,sha},
						  {ecdh_ecdsa,aes_256_gcm,null,sha384},
						  {ecdh_rsa,aes_256_gcm,null,sha384},
						  {dhe_rsa,aes_256_gcm,null,sha384},
						  {dhe_dss,aes_256_gcm,null,sha384},
						  {ecdh_ecdsa,aes_128_gcm,null,sha256},
						  {ecdh_rsa,aes_128_gcm,null,sha256},
						  {dhe_rsa,aes_128_gcm,null,sha256},
						  {dhe_dss,aes_128_gcm,null,sha256}]}
			  ]}
{config_entry_decoder, [
             {passphrase, <<"mypassphrase">>}
         ]}
```

config_entry_decoder也支持加载文件
```sh
{config_entry_decoder, [
             {passphrase, {file, "/path/to/passphrase/file"}}
         ]}
```

注意，encrypt_pwd必须是使用rabbitmq的encode命令来加密的
```sh
rabbitmqctl encode '<<"guest">>' mypassphrase
{encrypted,<<"... long encrypted value...">>}
rabbitmqctl encode '"decrypt_pwd"' mypassphrase
{encrypted,<<"... long encrypted value...">>}
```
rabbitmq的encode加密机制使用PBKDF2从密码生成一个派生密钥。默认的哈希函数为SHA512，默认的迭代次数为1000。默认cipher为AES 256 CBC。
```sh
[
  {rabbit, [
      ...
      {config_entry_decoder, [
             {passphrase, "mypassphrase"},
             {cipher, blowfish_cfb64},
             {hash, sha256},
             {iterations, 10000}
         ]}
    ]}
].
```

rabbitmq在启动加密配置文件的时候，会通过decode进行解密
```sh
rabbitmqctl decode '{encrypted, <<"encrypt_pwd">>}' mypassphrase
<<"decrypt_pwd">>
```

## 如何动态增加config参数
```python
import binascii
import copy
import json
import os
import shutil
import subprocess
from basesdk import utils
import tempfile

from kmc import kmc

RABBITMQCTL_PATH = "/usr/local/lib/rabbitmq/sbin/rabbitmqctl"
ENCRYPT_PWD_PATH = "/opt/rabbitmq/privkey/privkey.conf"
PASSPHARSE_PATH = "/etc/rabbitmq/passpharse"
RABBITMQ_CFG_PATH = "/usr/local/lib/rabbitmq/etc/rabbitmq/rabbitmq.config"

LOG = utils.get_logger("rabbitmq")


def get_private_key_pwd():
    if not os.path.exists(ENCRYPT_PWD_PATH):
        LOG.error("File({file}) is not found.".format(file=ENCRYPT_PWD_PATH))
        return

    try:
        with open(ENCRYPT_PWD_PATH, "r") as f:
            kmc_api = kmc.API()
            encrypt_pwd = f.read().split(" ")[1].rstrip('\n')
            decrypted_pwd = kmc_api.decrypt(0, encrypt_pwd)
    except Exception as err:
        LOG.exception("Decrypt password failed, "
                      "msg: {msg}".format(msg=err))
        return

    # Get secure random number as encryption key.
    random_num = get_random(64)
    if not random_num:
        return
    passpharse = binascii.hexlify(random_num).decode()

    # Use rabbitmqctl tool encryption to generate cipher text,
    # the private key password must use "".
    cmd_list = [RABBITMQCTL_PATH,
                "encode", '"%s"' % decrypted_pwd, passpharse]
    rst = execute_cmd(cmd_list)
    if not rst:
        return
    # Rabbitmqctl encrypted output is binary text.
    rst = rst.decode()

    # Encrypting value ...
    # {encrypted,<<"encrypted_text_here==">>}
    if not rst or "encrypted" not in rst:
        LOG.error("Failed to get encrypted text.")
        return

    # Write passpharse to temporary file.
    with open(PASSPHARSE_PATH, "w") as f:
        chown_cmd = "chown openstack:openstack {file}".format(
            file=PASSPHARSE_PATH)
        execute_cmd(chown_cmd.split(" "))
        chmod_pwd = "chmod 400 {file}".format(file=PASSPHARSE_PATH)
        execute_cmd(chmod_pwd.split(" "))
        f.write(str(passpharse))

    return rst.split("\n")[1]


def decryption_password(value):
    try:
        kmc_api = kmc.API()
        password = kmc_api.decrypt(0, value)
    except Exception as err:
        LOG.exception("Decryption password occur exc, "
                      "msg: {msg}".format(msg=err))
        return value
    else:
        return password.strip()


def execute_cmd(cmd, check_code=True):
    process = subprocess.Popen(cmd, stdout=subprocess.PIPE,
                               stdin=subprocess.PIPE, stderr=subprocess.PIPE)
    stdout, stderr = process.communicate()
    if process.returncode != 0 and check_code:
        LOG.error("An error occurred when executing "
                  "cmd. err：{msg}".format(msg=stderr))
    else:
        return stdout.strip()


def get_random(size=16, retry=3):
    for i in range(retry):
        try:
            with open("/dev/random", "rb") as f:
                return f.read(size)
        except Exception as err:
            LOG.exception("Get secure random failed, "
                          "msg: {msg}".format(msg=err))
    LOG.error("{num} attempts to get secure random number "
              "failed.".format(num=retry))


def read_file(file_name, mode='lines'):
    if not os.path.exists(file_name):
        LOG.error("File({file}) is not found.".format(file=file_name))
        if mode == 'lines':
            return []
        elif mode == 'json':
            return {}
        else:
            return ''

    with open(file_name, "r") as fd:
        if mode == 'lines':
            content = fd.readlines()
        elif mode == 'all':
            content = fd.read()
        elif mode == 'json':
            content = json.load(fd)
        else:
            content = ''

    return content


def write_file(file_name, content):
    fd = None
    try:
        fd, t_file_name = tempfile.mkstemp()
        os.write(fd, content)
        os.close(fd)
        shutil.move(t_file_name, file_name)
    except IOError:
        if fd:
            os.close(fd)
        LOG.error("Write content to file({file}) "
                  "failed.".format(file=file_name))


def main():
    old_content = read_file(RABBITMQ_CFG_PATH)
    if not old_content:
        exit(1)
    new_content = copy.deepcopy(old_content)

    encrypted_pwd = get_private_key_pwd()
    if not encrypted_pwd:
        exit(1)

    for line in old_content:
        if "{ssl_options" in line:
            # Config passphrase file path for decryption password.
            passphrase_line = '    {config_entry_decoder, ' \
                              '[{passphrase, {file, ' \
                              '"%s"}}]},\n' % PASSPHARSE_PATH
            index = new_content.index(line)
            new_content.insert(index, passphrase_line)
        elif "{keyfile" in line:
            # Config private key password.
            password_line = '                   ' \
                            '{password, %s},\n' % encrypted_pwd
            index = new_content.index(line)
            new_content.insert(index + 1, password_line)

    if old_content != new_content:
        write_file(RABBITMQ_CFG_PATH, "".join(new_content).encode())


if __name__ == '__main__':
    try:
        main()
    except Exception as err:
        LOG.exception("Update rabbitmq safe config failed, "
                      "msg: {msg}".format(msg=err))
        rm_file_cmd = "rm -f {file}".format(file=PASSPHARSE_PATH)
        execute_cmd(rm_file_cmd.split(" "))
        exit(1)
    else:
        LOG.info("Update rabbitmq safe config success.")
        exit(0)

```

## 生成加密私钥
```sh
#!/usr/bin/env bash

cert_path="/opt/rabbitmq/"
root_cert_path="/opt/rabbitmq/"
privkey_path="/opt//opt/rabbitmq/privkey/privkey.conf"

function get_private_key_pwd()
{
     encrypt_password=$(cat ${privkey_path} | awk '{print $2}')
     python3 -c "import kmc.kmc;A=kmc.kmc.API();print(A.decrypt(0,'$encrypt_password'))"
}


function pre_requisites()
{
    mkdir -p ${cert_path}/demoCA/
    mkdir -p ${cert_path}/demoCA/newcerts
    touch ${cert_path}/demoCA/index.txt
    local rand_serial=`openssl rand -base64 8 | md5sum | cut -c1-8`
    echo "${rand_serial}" > ${cert_path}/demoCA/serial
    echo "unique_subject = no" > ${cert_path}/demoCA/index.txt.attr
}


function create_key_and_cert()
{
    cd ${cert_path}
    local cert_name=rabbitmq
    local csr_cn=`get_info.py manage_float_ip`
    local pwd=$(get_private_key_pwd)

    openssl genrsa -passout pass:${pwd} -aes256 -out ${cert_path}/${cert_name}-key.pem 2048
    if [[ ! -a ${cert_path}/${cert_name}-key.pem ]]; then
        exit -1
    fi

    openssl req -new -sha256 -key ${cert_path}/${cert_name}-key.pem -out ${cert_path}/${cert_name}-req.pem -days 3650 -subj "/C=CN/ST=SiChuan/O=Huawei/CN=${csr_cn}" -passin pass:${pwd}
    if [[ ! -a ${cert_path}/${cert_name}-req.pem ]];then
        exit -1
    fi

    openssl ca -batch -in ${cert_path}/${cert_name}-req.pem -cert ${root_cert_path}/ca-cert.pem -keyfile ${root_cert_path}/ca-key.pem -out ${cert_path}/${cert_name}-cert.pem -md sha256 -days 3650 -passin pass:${pwd}
    if [[ ! -a ${cert_path}/${cert_name}-cert.pem ]];then
        exit -1
    fi
}

function main()
{
    pre_requisites
    create_key_and_cert
}

main
```