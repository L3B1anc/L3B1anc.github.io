---
layout: post
title:  "0x02 keychain&keystore"
date:   2019-10-16 11：16
categories: [Android]
tags: attack_me_if_you_can
---
<!-- more -->
# 0x02 keychain&keystore

> Attack me if you can level 2& level 3

## keychain证书安装机制

```java
private void installPkcs12() {
        try {
            for (String f1 : getAssets().list("")) {
                Log.v("names", f1);
            }
            BufferedInputStream bis = new BufferedInputStream(getAssets().open(PKCS12_FILENAME));
            byte[] keychain = new byte[bis.available()];
            bis.read(keychain);
            Intent installIntent = KeyChain.createInstallIntent();
            installIntent.putExtra("PKCS12", keychain);
            startActivity(installIntent);
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e2) {
            e2.printStackTrace();
        }
    }
```

## keystore读取密钥

从keystore导出生成rsa密钥对

```java
  public void createNewKeys() {
        String alias = "Dummy";
        try {
            if (this.keyStore.containsAlias(alias)) {
                Toast.makeText(getApplicationContext(), "Key Pair \"Dummy\" already created.", 1).show();
                return;
            }
            Calendar start = Calendar.getInstance();
            Calendar end = Calendar.getInstance();
            end.add(1, 1);
            KeyPairGeneratorSpec spec = null;
            if (VERSION.SDK_INT >= 18) {
                spec = new Builder(this).setAlias(alias).setSubject(new X500Principal("CN=Sample Name, O=Android Authority")).setSerialNumber(BigInteger.ONE).setStartDate(start.getTime()).setEndDate(end.getTime()).build();
            }
            KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA", "AndroidKeyStore");
            generator.initialize(spec);
            KeyPair keyPair = generator.generateKeyPair();
            Toast.makeText(getApplicationContext(), "Key Pair \"Dummy\" created.", 1).show();
        } catch (Exception e) {
            StringBuilder stringBuilder = new StringBuilder();
            stringBuilder.append("Exception ");
            stringBuilder.append(e.getMessage());
            stringBuilder.append(" occured");
            Toast.makeText(this, stringBuilder.toString(), 1).show();
            Log.e(this.TAG, Log.getStackTraceString(e));
        }
    }

```

利用rsa的密钥对进行加密解密
