---
title: "ANR Intro"
date: 2022-04-12T20:54:32+08:00
tags: ["anr"]
description: "ANR 简介和原理"
categories: ["android"]
author: "Zac"
draft: true
---

ANR 原理

### ANR 的触发场景

``` java
// com/android/server/am/ActivityManagerService.java
// 1.Broadcast 广播前台 10s 内未执行，后台 60s 内未执行
// How long we allow a receiver to run before giving up on it.
static final int BROADCAST_FG_TIMEOUT = 10*1000;
static final int BROADCAST_BG_TIMEOUT = 60*1000;
// 2.输入事件分发超过 5s，这里可以包括按键输入和触摸输入
// How long we wait until we timeout on key dispatching.
static final int KEY_DISPATCHING_TIMEOUT = 5*1000;

// com/android/server/am/ActiveServices.java
// 3.前台服务超过 20s 未执行，后台服务超过 200s 未执行
// How long we wait for a service to finish executing.
static final int SERVICE_TIMEOUT = 20*1000;
// How long we wait for a service to finish executing.
static final int SERVICE_BACKGROUND_TIMEOUT = SERVICE_TIMEOUT * 10;
```

### ANR 触发流程

以后台 Service 为例：

Context.startService()
-> AMS.startService()
-> ActiveServices.startService()
-> ActiveServices.realStartServiceLocked()

``` java
private final void realStartServiceLocked(ServiceRecord r,ProcessRecord app, boolean execInFg) throws RemoteException {
    if (app.thread == null) {
        throw new RemoteException();
    }
    if (DEBUG_MU)
        Slog.v(TAG_MU, "realStartServiceLocked, ServiceRecord.uid = " + r.appInfo.uid
                + ", ProcessRecord.uid = " + app.uid);
    r.app = app;
    r.restartTime = r.lastActivity = SystemClock.uptimeMillis();
    app.services.add(r);
    bumpServiceExecutingLocked(r, execInFg, "create");
    mAm.updateLruProcessLocked(app, false, null);
    mAm.updateOomAdjLocked();
    boolean created = false;
    try {
        String nameTerm;
        int lastPeriod = r.shortName.lastIndexOf('.');
        nameTerm = lastPeriod >= 0 ? r.shortName.substring(lastPeriod) : r.shortName;
        if (LOG_SERVICE_START_STOP) {
            EventLogTags.writeAmCreateService(
                    r.userId, System.identityHashCode(r), nameTerm, r.app.uid, r.app.pid);
        }
        synchronized (r.stats.getBatteryStats()) {
            r.stats.startLaunchedLocked();
        }
        mAm.ensurePackageDexOpt(r.serviceInfo.packageName);
        app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
        app.thread.scheduleCreateService(r, r.serviceInfo,
                mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                app.repProcState);
        r.postNotification();
        created = true;
    } catch (DeadObjectException e) {
        Slog.w(TAG, "Application dead when creating service " + r);
        mAm.appDiedLocked(app);
    } finally {
        if (!created) {
            app.services.remove(r);
            r.app = null;
            scheduleServiceRestartLocked(r, false);
            return;
        }
    }
    requestServiceBindingsLocked(r, execInFg);
    updateServiceClientActivitiesLocked(app, null, true);
    // If the service is in the started state, and there are no
    // pending arguments, then fake up one so its onStartCommand() will
    // be called.
    if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                null, null));
    }
    sendServiceArgsLocked(r, execInFg, true);
    if (r.delayed) {
        if (DEBUG_DELAYED_STARTS) Slog.v(TAG, "REM FR DELAY LIST (new proc): " + r);
        getServiceMap(r.userId).mDelayedStartList.remove(r);
        r.delayed = false;
    }
    if (r.delayedStop) {
        // Oh and hey we've already been asked to stop!
        r.delayedStop = false;
        if (r.startRequested) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG, "Applying delayed stop (from start): " + r);
            stopServiceLocked(r);
        }
    }
}
```
