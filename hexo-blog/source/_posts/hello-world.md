---
title: Hello World
toc: true
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## Quick Start

### Create a new post

``` bash
$ hexo new "My New Post"
```
<!-- more -->
More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)


```java
package com.hypers.task;

import com.alibaba.fastjson.JSON;
import com.hypers.S3.S3Config;
import com.hypers.S3.aws.AwsS3Upload;
import com.hypers.entity.Account;
import com.hypers.entity.Campaign;
import com.hypers.entity.ManualMessage;
import com.hypers.service.CampaignService;
import com.hypers.util.common.FileUtil;
import com.hypers.util.hfa.ZipUtils;
import java.io.File;
import java.io.FileInputStream;
import java.util.ArrayList;
import java.util.Calendar;
import java.util.Date;
import java.util.List;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

/**
 * 活动定时上传s3，
 */
@Component
public class CampaignTask {

  @Autowired
  AwsS3Upload upload;
  @Autowired
  S3Config s3Config;
  @Autowired
  private CampaignService campaignService;

  private static final String name = "FREQUENCY";
  private static final String SUB_JSON = ".json";
  private static final String SUB_ZIP = ".zip";
  private static final String TEMP = "/temp/";

  /**
   * 每天凌晨执行，统计已过去一天需计算的内容
   */
  @Scheduled(cron = "10 0 0 * * ? ")
  public void execute() {
    /**
     * 统计上传所需计算的当天截止时间的所有活动（出现过的活动,无视停启用状态）
     * TA 筛选的问题需要加
     */
    try {
      Calendar calendar = Calendar.getInstance(); // calendar.set(2019, 11, 26);
      Date date = calendar.getTime();//DateUtils.addDays(, );//DateUtils.addMonths( -1);

      uploadFrequencyFile(date, null, null);
    } catch (Exception e) {
      e.printStackTrace();
    }
  }

  /**
   * 频次计算清单文件上传
   */
  private void uploadFrequencyFile(Date date, List<Long> accountIds, List<Long> orderIds)
      throws Exception {
    String localPath = s3Config.getTempPath() + TEMP;
    List<Campaign> campaignByDate = campaignService
        .calCampaign(campaignService.getCampaignByDate(date, accountIds, orderIds));
    String content = JSON.toJSONString(campaignByDate);

    FileUtil.writeFile(localPath, name + SUB_JSON, content);
    File file = new File(localPath + name + SUB_JSON);
    ZipUtils.zipFile(file, "");
    file = new File(localPath + name + SUB_ZIP);
//    upload.uploadFile(s3Config.getBucket(), s3Config.getFrequencyPath(), file);
    upload.uploadFileWithPublicReadAccess(s3Config.getBucket(), s3Config.getFrequencyPath(),
        file.getName(), new FileInputStream(file), 0);
    FileUtil.delete(localPath, name);
  }

  /**
   * 人工操作：指定日期范围
   */
  public void execute(ManualMessage message) throws Exception {
    switch (message.getType()) {
      case "operate": // 运营报告
        //计算运营计算清单，发送计算端，目前不支持
        break;
      case "frequency": // 频次报告重算（测试操作）
        //计算重算清单，发送计算端
        /**
         * 1.仅指定天重算对应天数据，范围型不支持 (不支持)
         * 2.指定活动，重算该活动及其下所有订单 （没有时间范围默认全范围，可指定范围）
         * 3.指定订单，只算对应活动的指定订单（没有时间范围默认全范围，可指定范围）
         */
        List<Long> accountIds = new ArrayList<>();
        List<Long> lineOrderIds = new ArrayList<>();
        List<Account> account = message.getAccount();
        account.forEach(a -> {
          if (a.getLineOrders() != null && a.getLineOrders().size() > 0) {
            lineOrderIds.addAll(a.getLineOrders());
          } else {
            accountIds.add(a.getId());
          }
        });
        Calendar calendar = Calendar.getInstance(); // calendar.set(2019, 11, 26);
        Date date = calendar.getTime();//DateUtils.addDays(, -1);//DateUtils.addMonths( -1);
        uploadFrequencyFile(date, accountIds, lineOrderIds);
        break;
      default:
        // 发送报警邮件
        break;
    }
  }


}


```
