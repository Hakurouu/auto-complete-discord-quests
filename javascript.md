```javascript
(async () => {
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
        Channels: findByProp('getAllThreadsForParent'),
        Guilds: findByExport('getSFWDefaultChannel')
    };

    const TASKS = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"];
    const isNative = typeof DiscordNative !== "undefined";

    const getActiveQuests = () => [...Services.Quests.quests.values()].filter(q => 
        q.userStatus?.enrolledAt && 
        !q.userStatus?.completedAt && 
        new Date(q.config.expiresAt) > Date.now() &&
        TASKS.some(t => (q.config.taskConfigV2 ?? q.config.taskConfig).tasks[t])
    );

    const sleep = (ms) => new Promise(res => setTimeout(res, ms));

    const executeQuest = async (quest) => {
        const config = quest.config.taskConfigV2 ?? quest.config.taskConfig;
        const taskType = TASKS.find(t => config.tasks[t]);
        const target = config.tasks[taskType].target;
        const appId = quest.config.application.id;
        let progress = quest.userStatus?.progress?.[taskType]?.value ?? 0;

        if (taskType.includes("WATCH_VIDEO")) {
            console.log(`%c[Video] Spoofing: ${quest.config.messages.questName}`, "color: #00b0f4");
            while (progress < target) {
                const step = 7 + Math.random();
                const res = await Services.API.post({
                    url: `/quests/${quest.id}/video-progress`,
                    body: { timestamp: Math.min(target, progress + step) }
                });
                progress = Math.min(target, progress + step);
                if (res.body.completed_at) break;
                await sleep(1500);
            }
        } 
        
        else if (taskType === "PLAY_ON_DESKTOP" && isNative) {
            const pid = Math.floor(Math.random() * 30000) + 1000;
            const originalGetGames = Services.RunningGames.getRunningGames;
            const originalGetPID = Services.RunningGames.getGameForPID;
            
            const mockGame = {
                id: appId,
                name: quest.config.application.name,
                pid: pid,
                start: Date.now(),
                exePath: "c:/fake_path/game.exe"
            };

            Services.RunningGames.getRunningGames = () => [mockGame];
            Services.RunningGames.getGameForPID = () => mockGame;
            
            Services.Dispatcher.dispatch({
                type: "RUNNING_GAMES_CHANGE", 
                removed: [], 
                added: [mockGame], 
                games: [mockGame]
            });

            await new Promise(resolve => {
                const handler = (data) => {
                    const cur = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : data.userStatus.progress.PLAY_ON_DESKTOP.value;
                    if (cur >= target) {
                        Services.Dispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", handler);
                        Services.RunningGames.getRunningGames = originalGetGames;
                        Services.RunningGames.getGameForPID = originalGetPID;
                        resolve();
                    }
                };
                Services.Dispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", handler);
            });
        }

        else if (taskType === "STREAM_ON_DESKTOP" && isNative) {
            const originalMeta = Services.Streaming.getStreamerActiveStreamMetadata;
            Services.Streaming.getStreamerActiveStreamMetadata = () => ({ id: appId, pid: 1337, sourceName: null });

            await new Promise(resolve => {
                const handler = (data) => {
                    const cur = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : data.userStatus.progress.STREAM_ON_DESKTOP.value;
                    if (cur >= target) {
                        Services.Dispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", handler);
                        Services.Streaming.getStreamerActiveStreamMetadata = originalMeta;
                        resolve();
                    }
                };
                Services.Dispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", handler);
            });
        }

        else if (taskType === "PLAY_ACTIVITY") {
            const chanId = Services.Channels.getSortedPrivateChannels()[0]?.id;
            while (progress < target) {
                const res = await Services.API.post({
                    url: `/quests/${quest.id}/heartbeat`,
                    body: { stream_key: `call:${chanId}:1`, terminal: false }
                });
                progress = res.body.progress.PLAY_ACTIVITY.value;
                await sleep(20000);
            }
            await Services.API.post({ url: `/quests/${quest.id}/heartbeat`, body: { stream_key: `call:${chanId}:1`, terminal: true }});
        }
    };

    const list = getActiveQuests();
    for (const q of list) {
        await executeQuest(q);
        console.log(`%c[Done] ${q.config.messages.questName}`, "color: #43b581");
    }
})();
```
