---
title: AsyncTask基础复习
tags:
  - AsyncTask
categories:
  - Android
  - 基础知识
abbrlink: 7096
date: 2017-09-23 15:36:17
---

## AsyncTask

### 1、使用注意事项
* AsyncTask的实例必须在主线程中创建
* AsyncTask的execute方法必须在主线程中调用
* 回调方法，Android会自动调用 
* 一个AsyncTask的实例，只能执行一次execute方法
``` java

public class UpdateTextTask extends AsyncTask<Void, Integer, Integer> {
    private Context context;

    UpdateTextTask(Context context) {
        this.context = context;
    }
    
    /**
     * 运行在UI线程中，在调用doInBackground()之前执行
     */
    @Override
    protected void onPreExecute() {
        Toast.makeText(context, "开始执行", Toast.LENGTH_SHORT).show();
    }
    /**
     * 后台运行的方法，可以运行非UI线程，可以执行耗时的方法
     */
    @Override
    protected Integer doInBackground(Void... voids) {
        int i = 0;
        while (i < 10) {
            i++;
            publishProgress(i);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
            }
        }
        return null;
    }
    
    /**
     * 运行在ui线程中，在doInBackground()执行完毕后执行
     */
    @Override
    protected void onPostExecute(Integer integer) {
        Toast.makeText(context, "执行完毕", Toast.LENGTH_SHORT).show();
    }
    /**
     * 在publishProgress()被调用以后执行，publishProgress()用于更新进度
     */
    @Override
    protected void onProgressUpdate(Integer... values) {
//        tv.setText(""+values[0]);
    }

//使用
        new UpdateTextTask(MainActivity.this).execute();

```

