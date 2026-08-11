```(async () => {
    const webpackModules = Object.values(webpackChunkdiscord_app.push([[Symbol()], {}, r => r]).c);
    webpackChunkdiscord_app.pop();

    const findByProp = (p) => webpackModules.find(m => m?.exports?.A?.__proto__?.[p])?.exports?.A;
    const findByExport = (p) => webpackModules.find(m => m?.exports?.Ay?.[p])?.exports?.Ay;

    const Services = {
        Streaming: findByProp('getStreamerActiveStreamMetadata'),
        RunningGames: findByExport('getRunningGames'),
        Quests: findByProp('getQuest'),
        Dispatcher: webpackModules.find(m => m?.exports?.h?.__proto__?.flushWaitQueue)?.exports.h,
        API: webpackModules.find(m => m?.exports?.Bo?.get)?.exports.Bo,
        Channels: findByProp('getAllThreadsForParent')
    };

    const TASKS = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"];

    const getActiveQuests = () => [...Services.Quests.quests.values()].filter(q => 
        q.userStatus?.enrolledAt && 
        !q.userStatus?.completedAt && 
        new Date(q.config.expiresAt) > Date.now() &&
        TASKS.some(t => (q.config.taskConfigV2 ?? q.config.taskConfig)?.tasks?.[t])
    );

    const sleep = (ms) => new Promise(res => setTimeout(res, ms));

    const executeQuest = async (quest) => {
        const config = quest.config.taskConfigV2 ?? quest.config.taskConfig;
        const taskType = TASKS.find(t => config?.tasks?.[t]);
        
        if (!taskType) return;

        const target = config.tasks[taskType].target;
        let progress = quest.userStatus?.progress?.[taskType]?.value ?? 0;

        console.log(`%c[Quest] Starting: ${quest.config.messages?.questName} (${taskType})`, "color: #00b0f4");

        if (taskType.includes("WATCH_VIDEO")) {
            while (progress < target) {
                const step = 7 + Math.random();
                const res = await Services.API.post({
                    url: `/quests/${quest.id}/video-progress`,
                    body: { timestamp: Math.min(target, progress + step) }
                });
                progress = Math.min(target, progress + step);
                if (res?.body?.completed_at) break;
                await sleep(1500);
            }
        } 
        
        else if (taskType === "PLAY_ON_DESKTOP") {
            while (progress < target) {
                const res = await Services.API.post({
                    url: `/quests/${quest.id}/heartbeat`,
                    body: { stream_key: null, terminal: false }
                });
                
                progress = res?.body?.user_status?.progress?.PLAY_ON_DESKTOP?.value 
                        ?? res?.body?.progress?.PLAY_ON_DESKTOP?.value 
                        ?? (progress + 20);

                console.log(`%c[Progress] ${quest.config.messages?.questName}: ${progress}/${target}s`, "color: #e91e63");
                if (progress >= target) break;
                await sleep(20000);
            }

            await Services.API.post({
                url: `/quests/${quest.id}/heartbeat`,
                body: { stream_key: null, terminal: true }
            });
        }

        else if (taskType === "STREAM_ON_DESKTOP") {
            const originalMeta = Services.Streaming.getStreamerActiveStreamMetadata;
            const appId = quest.config.application?.id ?? quest.config.applicationId;

            try {
                Services.Streaming.getStreamerActiveStreamMetadata = () => ({ id: appId, pid: 1337, sourceName: null });

                await new Promise(resolve => {
                    const handler = (data) => {
                        if (data?.questId && data.questId !== quest.id) return;

                        const cur = quest.config.configVersion === 1 
                            ? data?.userStatus?.streamProgressSeconds 
                            : data?.userStatus?.progress?.STREAM_ON_DESKTOP?.value;

                        if (cur !== undefined && cur >= target) {
                            Services.Dispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", handler);
                            resolve();
                        }
                    };
                    Services.Dispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", handler);
                });
            } finally {
                Services.Streaming.getStreamerActiveStreamMetadata = originalMeta;
            }
        }

        else if (taskType === "PLAY_ACTIVITY") {
            const chanId = Services.Channels.getSortedPrivateChannels()[0]?.id;
            while (progress < target) {
                const res = await Services.API.post({
                    url: `/quests/${quest.id}/heartbeat`,
                    body: { stream_key: `call:${chanId}:1`, terminal: false }
                });
                progress = res?.body?.progress?.PLAY_ACTIVITY?.value ?? (progress + 20);
                await sleep(20000);
            }
            await Services.API.post({ url: `/quests/${quest.id}/heartbeat`, body: { stream_key: `call:${chanId}:1`, terminal: true }});
        }
    };

    const list = getActiveQuests();
    console.log(`[Quests] Found ${list.length} active quests.`);

    for (const q of list) {
        try {
            await executeQuest(q);
            console.log(`%c[Done] ${q.config.messages?.questName}`, "color: #43b581");
        } catch (err) {
            console.error(`[Error] Failed quest ${q.config.messages?.questName}:`, err);
        }
        await sleep(2000);
    }
})();
    }
})();
```
