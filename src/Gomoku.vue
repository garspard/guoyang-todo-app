<script setup>
import { ref, computed } from 'vue';

// --- 1. 基础配置 ---
const boardSize = 15;
const board = ref(Array(boardSize).fill(0).map(() => Array(boardSize).fill(0))); 
const currentPlayer = ref(1); 
const isGameOver = ref(false); 
const gameStatusText = ref('请开始游戏');
const winnerInfo = ref(null); 

// --- 2. 玩家配置 ---
const showSetup = ref(true);
const showTutorial = ref(false);
const showWinModal = ref(false);
const playerColor = ref(1);
const aiColor = computed(() => (playerColor.value === 1 ? 2 : 1));

// --- 3. 技能系统 ---
const skillUsing = ref(false);
const skillBlocked = ref(false); 
const aiSkillUsing = ref(false);
const playerSkills = ref({ 1: {}, 2: {} }); 
const globalUsedSkills = ref(new Set()); 

// --- 4. 技能库定义 (🌟 新增：tip 攻略字段) ---
const ALL_SKILLS = [
    { 
        key: 'flySand', name: '飞沙走石', maxCd: 4, type: 'offensive', minOpponentPieces: 1,
        desc: '随机移除对手 1 颗棋子。', 
        tip: '适合破防。当对手防守严密但没有形成连珠时，用运气打破僵局。'
    },
    { 
        key: 'timeStop', name: '静如止水', maxCd: 6, type: 'control',
        desc: '本回合对手无法落子，您可以连续行动两回合。', 
        tip: '最强杀招！当您有“活三”时开启，即可瞬间形成“活四”获胜。'
    },
    { 
        key: 'breakBoard', name: '力拔山兮', maxCd: 99, type: 'ultimate', minTotalPieces: 10, isGlobalOneTime: true,
        desc: '清空整个棋盘。全局双方限用一次，需场上>10子。', 
        tip: '绝境翻盘神技。当对手必胜且无法阻挡时，直接掀桌子重来！'
    },
    { 
        key: 'pickGold', name: '拾金不昧', maxCd: 5, type: 'defensive',
        desc: '进入防御状态，自动抵消对手下一次使用的技能。', 
        tip: '预判神技。当感觉电脑要放“静如止水”或“水滴石穿”时提前开启。'
    },
    { 
        key: 'dropWater', name: '水滴石穿', maxCd: 3, type: 'offensive', minOpponentPieces: 1,
        desc: '移除对手连成 3 子的棋子（优先移除威胁最大的）。', 
        tip: '防守神技。专门用来破坏对手的“活三”或“冲四”，冷却快，要舍得用。'
    },
    { 
        key: 'cleaner', name: '保洁上门', maxCd: 5, type: 'offensive', minTotalPieces: 5,
        desc: '清除棋盘上随机 3 颗棋子（不分敌我）。', 
        tip: '搅局技能。当局面混乱对自己不利时使用，增加变数。'
    },
    { 
        key: 'swap', name: '移花接木', maxCd: 4, type: 'chaos', minMyPieces: 1,
        desc: '随机将自己 1 颗子变成对方颜色。', 
        tip: '负面/混沌技能。通常是副作用，但也可能用来破坏对方想填入的关键位。'
    },
];

const skillEffectMap = {
    flySand: useFlySand,
    timeStop: useTimeStop,
    breakBoard: useBreakBoard,
    pickGold: usePickGold,
    dropWater: useDropWater,
    cleaner: useCleaner,
    swap: useSwap,
};

// --- 5. 游戏流程 ---

function startGame(choice) {
    board.value = Array(boardSize).fill(0).map(() => Array(boardSize).fill(0));
    isGameOver.value = false;
    showWinModal.value = false;
    skillUsing.value = false;
    skillBlocked.value = false;
    aiSkillUsing.value = false;
    currentPlayer.value = 1;
    globalUsedSkills.value.clear();

    if (choice === 'random') {
        playerColor.value = Math.random() > 0.5 ? 1 : 2;
    } else {
        playerColor.value = choice;
    }

    assignFixedSkills();
    showSetup.value = false;
    gameStatusText.value = `游戏开始！您执${playerColor.value === 1 ? '黑棋' : '白棋'}，技能已同步！`;

    if (playerColor.value === 2) {
        setTimeout(aiMove, 1000);
    }
}

function assignFixedSkills() {
    const shuffled = [...ALL_SKILLS].sort(() => 0.5 - Math.random());
    const matchSkills = shuffled.slice(0, 3); 
    playerSkills.value = { 1: {}, 2: {} };
    matchSkills.forEach(skill => {
        playerSkills.value[1][skill.key] = { ...skill, cd: 0 };
        playerSkills.value[2][skill.key] = { ...skill, cd: 0 };
    });
}

function switchTurn() {
    if (isGameOver.value) return;
    const nextPlayer = (currentPlayer.value === 1) ? 2 : 1;
    reduceCooldowns(nextPlayer);
    currentPlayer.value = nextPlayer;
    
    const isPlayerTurn = currentPlayer.value === playerColor.value;
    gameStatusText.value = isPlayerTurn ? `轮到您了！` : `轮到电脑...`;

    if (!isPlayerTurn) {
        setTimeout(aiMove, 800);
    }
}

function reduceCooldowns(player) {
    const skills = playerSkills.value[player];
    Object.keys(skills).forEach(key => {
        if (skills[key].cd > 0) skills[key].cd--;
    });
}

// --- 6. AI 逻辑 ---

function aiMove() {
    if (isGameOver.value || currentPlayer.value !== aiColor.value) return;

    const mySkills = playerSkills.value[aiColor.value];
    const playerThreat = assessThreat(playerColor.value);
    const myAttack = assessThreat(aiColor.value);

    // 1. 必杀判定
    if (myAttack.level >= 3 && mySkills['timeStop'] && mySkills['timeStop'].cd === 0) {
        aiUseSkill('timeStop'); return;
    }

    // 2. 危机处理
    let panicMode = false;
    if (playerThreat.level >= 4) panicMode = true; 
    else if (playerThreat.level === 3) {
        const bestDefensiveMove = getBestMove(true); 
        if (!bestDefensiveMove) panicMode = true; 
    }

    if (panicMode) {
        for (const key in mySkills) {
            const skill = mySkills[key];
            if (skill.cd === 0 && checkSkillCondition(skill, aiColor.value).ok) {
                if (['offensive', 'control', 'ultimate'].includes(skill.type)) {
                    aiUseSkill(key); return;
                }
            }
        }
    }

    // 3. 随机尝试
    if (Math.random() > 0.95 && !aiSkillUsing.value && playerThreat.level < 3) {
        const availableKeys = Object.keys(mySkills).filter(k => 
            mySkills[k].cd === 0 && 
            checkSkillCondition(mySkills[k], aiColor.value).ok &&
            mySkills[k].type !== 'ultimate'
        );
        if (availableKeys.length > 0) {
            const randomKey = availableKeys[Math.floor(Math.random() * availableKeys.length)];
            aiUseSkill(randomKey); return;
        }
    }

    // 4. 落子
    let bestMove = getBestMove();
    if (bestMove) {
        setTimeout(() => {
            if (isGameOver.value) return;
            board.value[bestMove.r][bestMove.c] = aiColor.value;
            if (!checkWin(bestMove.r, bestMove.c)) {
                switchTurn(); 
            }
        }, 800);
    } else {
        forceRandomMove();
    }
}

function aiUseSkill(key) {
    aiSkillUsing.value = true;
    gameStatusText.value = `🤖 电脑发动技能：${playerSkills.value[aiColor.value][key].name}`;
    setTimeout(() => { triggerAISkill(key); }, 1500);
}

function getBestMove(defensiveOnly = false) {
    let bestMove = null;
    let maxScore = -Infinity;

    for(let r=0; r<boardSize; r++) {
        for(let c=0; c<boardSize; c++) {
            if (board.value[r][c] === 0) {
                if (hasNeighbor(r,c) || (r===7 && c===7)) {
                    const attackScore = evaluatePoint(r, c, aiColor.value); 
                    const defenseScore = evaluatePoint(r, c, playerColor.value); 
                    
                    let totalScore = 0;
                    if (defensiveOnly) {
                        totalScore = defenseScore;
                    } else {
                        if (attackScore >= 100000) totalScore = 999999; 
                        else if (defenseScore >= 50000) totalScore = 500000 + attackScore; 
                        else if (defenseScore >= 5000) totalScore = 10000 + attackScore + defenseScore; 
                        else totalScore = attackScore + defenseScore;
                        totalScore += Math.random() * 10; 
                    }

                    if (totalScore > maxScore) {
                        maxScore = totalScore;
                        bestMove = {r, c};
                    }
                }
            }
        }
    }
    return bestMove;
}

function assessThreat(color) {
    let maxThreat = 0;
    for(let r=0; r<boardSize; r++) {
        for(let c=0; c<boardSize; c++) {
            if (board.value[r][c] === 0 && hasNeighbor(r,c)) {
                const score = evaluatePoint(r, c, color);
                if (score >= 100000) return { level: 5 }; 
                if (score >= 10000) return { level: 4 }; 
                if (score >= 1000) { if(maxThreat < 3) maxThreat = 3; }
            }
        }
    }
    return { level: maxThreat };
}

function evaluatePoint(r, c, color) {
    let score = 0;
    const directions = [[0, 1], [1, 0], [1, 1], [1, -1]]; 
    for (const [dr, dc] of directions) {
        let count = 1; let emptySides = 0; 
        for (const sign of [1, -1]) {
            let i = 1;
            while(true) {
                const nr = r + dr * i * sign; const nc = c + dc * i * sign;
                if (nr < 0 || nr >= boardSize || nc < 0 || nc >= boardSize) break;
                if (board.value[nr][nc] === color) count++;
                else if (board.value[nr][nc] === 0) { emptySides++; break; }
                else break;
                i++;
            }
        }
        if (count >= 5) score += 100000;
        else if (count === 4) { if (emptySides === 2) score += 50000; else if (emptySides === 1) score += 10000; }
        else if (count === 3) { if (emptySides === 2) score += 5000; else if (emptySides === 1) score += 1000; }
        else if (count === 2) { if (emptySides === 2) score += 500; }
    }
    return score;
}

function hasNeighbor(r, c) {
    for(let i=-1; i<=1; i++) for(let j=-1; j<=1; j++) {
        if (i===0 && j===0) continue;
        const nr=r+i, nc=c+j;
        if(nr>=0 && nr<boardSize && nc>=0 && nc<boardSize && board.value[nr][nc]!==0) return true;
    }
    return false;
}

function forceRandomMove() {
    for(let r=0; r<boardSize; r++) for(let c=0; c<boardSize; c++) 
        if(board.value[r][c]===0) { board.value[r][c] = aiColor.value; switchTurn(); return; }
}

// --- 7. 条件与辅助 ---
function countTotalPieces() {
    let count = 0;
    for(let r=0; r<boardSize; r++) for(let c=0; c<boardSize; c++) if(board.value[r][c]!==0) count++;
    return count;
}
function countPlayerPieces(color) {
    let count = 0;
    for(let r=0; r<boardSize; r++) for(let c=0; c<boardSize; c++) if(board.value[r][c]===color) count++;
    return count;
}

function checkSkillCondition(skill, userColor) {
    if (skill.isGlobalOneTime && globalUsedSkills.value.has(skill.key)) return { ok: false, msg: '全局限一次' };
    if (skill.minTotalPieces && countTotalPieces() < skill.minTotalPieces) return { ok: false, msg: `需>${skill.minTotalPieces}子` };
    if (skill.minOpponentPieces) {
        const oppColor = userColor === 1 ? 2 : 1;
        if (countPlayerPieces(oppColor) < skill.minOpponentPieces) return { ok: false, msg: '无目标' };
    }
    if (skill.minMyPieces) {
        if (countPlayerPieces(userColor) < skill.minMyPieces) return { ok: false, msg: '无己方子' };
    }
    return { ok: true };
}

// --- 8. 触发逻辑 ---

function triggerPlayerSkill(key) {
    if (isGameOver.value || currentPlayer.value !== playerColor.value || skillUsing.value) return;
    
    const skill = playerSkills.value[playerColor.value][key];
    if (skill.cd > 0) { alert(`冷却中！还需 ${skill.cd} 回合`); return; }
    const condition = checkSkillCondition(skill, playerColor.value);
    if (!condition.ok) { alert(condition.msg); return; }

    skillUsing.value = true;
    if (skill.isGlobalOneTime) globalUsedSkills.value.add(key);

    skillEffectMap[key](playerColor.value, aiColor.value);
    skill.cd = skill.maxCd;

    setTimeout(() => {
        skillUsing.value = false;
        if (key !== 'timeStop') {
            switchTurn();
        } else {
            gameStatusText.value = '静如止水生效！您继续行动';
        }
    }, 1500);
}

function triggerAISkill(key) {
    const skill = playerSkills.value[aiColor.value][key];
    if (skill.isGlobalOneTime) globalUsedSkills.value.add(key);

    skillEffectMap[key](aiColor.value, playerColor.value);
    skill.cd = skill.maxCd; 

    setTimeout(() => {
        aiSkillUsing.value = false;
        if (key !== 'timeStop') {
            switchTurn();
        } else {
            gameStatusText.value = '电脑发动静如止水，继续行动...';
            setTimeout(aiMove, 1000);
        }
    }, 1500);
}

// --- 9. 技能效果 ---
function checkBlock() {
    if (skillBlocked.value) {
        gameStatusText.value = '❌ 技能被拾金不昧抵消！';
        skillBlocked.value = false;
        return true;
    }
    return false;
}

function useFlySand(user, opp) {
    if (checkBlock()) return;
    const targets = [];
    for(let r=0; r<boardSize; r++) for(let c=0; c<boardSize; c++) if(board.value[r][c]===opp) targets.push({r,c});
    if(targets.length){
        const t = targets[Math.floor(Math.random()*targets.length)];
        board.value[t.r][t.c] = 0;
        gameStatusText.value = `🌪️ 飞沙走石！移除了一颗棋子`;
    }
}
function useTimeStop(user) { if (checkBlock()) return; gameStatusText.value = `⏳ 静如止水！`; }
function useBreakBoard(user) { if (checkBlock()) return; board.value = board.value.map(row => row.map(()=>0)); gameStatusText.value = `💥 棋盘清空！`; }
function usePickGold(user) { skillBlocked.value = true; gameStatusText.value = `💰 拾金不昧开启！`; }
function useDropWater(user, opp) { 
    if (checkBlock()) return; 
    const targets = [];
    for(let r=0; r<boardSize; r++) for(let c=0; c<boardSize; c++) {
        if (board.value[r][c] === opp) {
            if (evaluatePoint(r,c, opp) > 500) targets.push({r,c});
        }
    }
    if (targets.length > 0) {
        const t = targets[Math.floor(Math.random()*targets.length)];
        board.value[t.r][t.c] = 0;
    } else {
        useFlySand(user, opp);
    }
    gameStatusText.value = `💧 水滴石穿！`; 
}
function useCleaner(user) {
    if (checkBlock()) return;
    let count = 0;
    for(let i=0; i<3; i++) {
        const r = Math.floor(Math.random()*boardSize), c = Math.floor(Math.random()*boardSize);
        if(board.value[r][c]!==0) { board.value[r][c]=0; count++; }
    }
    gameStatusText.value = `🧹 保洁上门带走了${count}颗子`;
}
function useSwap(user) {
    if (checkBlock()) return;
    for(let r=0; r<boardSize; r++) for(let c=0; c<boardSize; c++) {
        if(board.value[r][c] === user) { board.value[r][c] = (user===1?2:1); gameStatusText.value = `🤡 移花接木...棋子变色了！`; return; }
    }
}

// 玩家落子
function handlePlayerMove(r, c) {
    if (isGameOver.value || currentPlayer.value !== playerColor.value || board.value[r][c] !== 0 || skillUsing.value) return;
    board.value[r][c] = playerColor.value;
    if (!checkWin(r, c)) { switchTurn(); }
}

function checkWin(r, c) {
  const piece = board.value[r][c];
  if (!piece) return false;
  const directions = [ [0, 1], [1, 0], [1, 1], [1, -1] ];
  for (const [dr, dc] of directions) {
    let count = 1; 
    for (let i = 1; i < 5; i++) {
      const nr = r + dr * i; const nc = c + dc * i;
      if (nr >= 0 && nr < boardSize && nc >= 0 && nc < boardSize && board.value[nr][nc] === piece) count++; else break;
    }
    for (let i = 1; i < 5; i++) {
      const nr = r - dr * i; const nc = c - dc * i;
      if (nr >= 0 && nr < boardSize && nc >= 0 && nc < boardSize && board.value[nr][nc] === piece) count++; else break;
    }
    if (count >= 5) {
      const winnerName = piece === playerColor.value ? '恭喜获胜！🎉' : '遗憾落败...🤖';
      isGameOver.value = true;
      winnerInfo.value = {
          title: winnerName,
          color: piece === 1 ? '黑棋' : '白棋',
          desc: piece === playerColor.value ? '你的策略无懈可击！' : 'AI 的计算略胜一筹。'
      };
      showWinModal.value = true;
      return true;
    }
  }
  return false;
}
</script>

<template>
  <div class="game-container">
    <div class="header">
        <h2>⚔️ 智能技能五子棋</h2>
        <button class="tutorial-btn" @click="showTutorial = true">📖 技能图鉴</button>
    </div>

    <div v-if="showSetup" class="modal-overlay">
        <div class="modal-content">
            <h3>🎮 游戏设置</h3>
            <div class="btn-group">
                <button @click="startGame(1)">👤 执黑 (先手)</button>
                <button @click="startGame(2)">🤖 执白 (后手)</button>
                <button @click="startGame('random')">🎲 随机分配</button>
            </div>
        </div>
    </div>

    <div v-if="showWinModal" class="modal-overlay">
        <div class="modal-content win-content">
            <h3>{{ winnerInfo.title }}</h3>
            <p class="win-detail">胜方：{{ winnerInfo.color }}</p>
            <p>{{ winnerInfo.desc }}</p>
            <div class="btn-group">
                <button class="primary-btn" @click="showSetup = true">🏆 再来一局</button>
                <button class="secondary-btn" @click="showWinModal = false">👀 查看棋盘</button>
            </div>
        </div>
    </div>

    <div v-if="showTutorial" class="modal-overlay" @click.self="showTutorial = false">
        <div class="modal-content tutorial-content">
            <h3>📖 技能与规则解析</h3>
            
            <div class="rules-box">
                <h4>📜 基本规则</h4>
                <p>1. 率先连成5子获胜。</p>
                <p>2. 双方技能完全相同，CD独立计算。</p>
            </div>

            <div class="skill-analysis-box">
                <h4>✨ 全技能深度解析</h4>
                <ul class="analysis-list">
                    <li v-for="skill in ALL_SKILLS" :key="skill.key" :class="skill.type">
                        <div class="sk-header">
                            <span class="sk-name">{{ skill.name }}</span>
                            <span class="sk-cd">冷却: {{ skill.maxCd }}回合</span>
                        </div>
                        <div class="sk-desc">{{ skill.desc }}</div>
                        <div class="sk-tip">💡 攻略: {{ skill.tip }}</div>
                    </li>
                </ul>
            </div>
            
            <button class="close-btn" @click="showTutorial = false">我已了解</button>
        </div>
    </div>

    <div class="main-game" v-if="!showSetup">
        
        <div class="skills-section ai-skills">
            <h4 class="section-title">🤖 电脑技能 (CD监控)</h4>
            <div class="skills-grid">
                <div v-for="(skill, key) in playerSkills[aiColor]" :key="key"
                     class="skill-card ai-card"
                     :class="{ 'on-cd': skill.cd > 0 }">
                    <div class="s-name">{{ skill.name }}</div>
                    <div class="s-info" v-if="skill.cd > 0">CD: {{ skill.cd }}</div>
                    <div class="s-info ready" v-else>Ready</div>
                </div>
            </div>
        </div>

        <div class="status-bar">
            <p>{{ gameStatusText }}</p>
        </div>

        <div class="board-wrapper">
            <div class="board">
                <div v-for="(row, r) in board" :key="r" class="board-row">
                    <div v-for="(cell, c) in row" :key="c" class="cell" @click="handlePlayerMove(r, c)">
                        <div v-if="cell === 1" class="piece black"></div>
                        <div v-if="cell === 2" class="piece white"></div>
                    </div>
                </div>
            </div>
        </div>

        <div class="skills-section player-skills" :class="{ disabled: currentPlayer !== playerColor }">
            <h4 class="section-title">👤 您的技能</h4>
            <div class="skills-grid">
                <div v-for="(skill, key) in playerSkills[playerColor]" :key="key"
                     class="skill-card"
                     :class="{ 
                        'on-cd': skill.cd > 0,
                        'condition-fail': !checkSkillCondition(skill, playerColor).ok
                     }"
                     @click="triggerPlayerSkill(key)">
                    <div class="s-name">{{ skill.name }}</div>
                    <div class="s-desc">{{ skill.desc }}</div>
                    <div class="s-info" v-if="skill.cd > 0">冷却: {{ skill.cd }}</div>
                    <div class="s-info condition" v-else-if="!checkSkillCondition(skill, playerColor).ok">
                        {{ checkSkillCondition(skill, playerColor).msg }}
                    </div>
                    <div class="s-info ready" v-else>点击使用</div>
                </div>
            </div>
        </div>

        <button class="restart-btn" @click="showSetup = true">⚙️ 重新开始</button>
    </div>
  </div>
</template>

<style scoped>
/* 全局样式 */
.game-container { font-family: 'Segoe UI', sans-serif; max-width: 500px; margin: 0 auto; padding: 10px; }
.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 5px; }
.tutorial-btn { background: #3498db; color: white; border: none; padding: 6px 12px; border-radius: 4px; cursor: pointer; font-size: 14px; }

/* 弹窗通用 */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.6); display: flex; justify-content: center; align-items: center; z-index: 100; backdrop-filter: blur(2px); }
.modal-content { background: white; padding: 20px; border-radius: 12px; width: 320px; text-align: center; box-shadow: 0 10px 30px rgba(0,0,0,0.3); animation: popIn 0.3s ease; max-height: 85vh; overflow-y: auto; }
@keyframes popIn { from { transform: scale(0.9); opacity: 0; } to { transform: scale(1); opacity: 1; } }

/* 🌟 教程样式 */
.tutorial-content { text-align: left; width: 420px; }
.rules-box { background: #f8f9fa; padding: 10px; border-radius: 8px; margin-bottom: 15px; border-left: 4px solid #3498db; }
.rules-box p { margin: 4px 0; font-size: 13px; color: #555; }
.skill-analysis-box h4 { margin-bottom: 10px; color: #2c3e50; border-bottom: 1px solid #eee; padding-bottom: 5px; }
.analysis-list { list-style: none; padding: 0; }
.analysis-list li { margin-bottom: 12px; border: 1px solid #eee; padding: 8px; border-radius: 6px; background: #fff; }
.sk-header { display: flex; justify-content: space-between; margin-bottom: 4px; }
.sk-name { font-weight: bold; color: #d35400; }
.sk-cd { font-size: 11px; color: #999; background: #f0f0f0; padding: 2px 5px; border-radius: 4px; }
.sk-desc { font-size: 12px; color: #333; margin-bottom: 4px; }
.sk-tip { font-size: 11px; color: #27ae60; background: #eafaf1; padding: 4px; border-radius: 4px; }

/* 游戏 UI */
.status-bar { background: #f0f0f0; padding: 8px; border-radius: 5px; margin-bottom: 10px; text-align: center; font-weight: bold; color: #2c3e50; }
.skills-section { margin-bottom: 10px; transition: opacity 0.3s; background: #fff; padding: 5px; border-radius: 8px; border: 1px solid #eee; }
.skills-section.disabled { opacity: 0.6; pointer-events: none; }
.section-title { font-size: 12px; color: #999; margin-bottom: 5px; text-align: left; padding-left: 5px; }
.skills-grid { display: flex; gap: 8px; justify-content: space-between; }

.skill-card {
    background: #fff; border: 1px solid #e67e22; border-radius: 6px;
    width: 32%; padding: 6px; cursor: pointer; box-shadow: 0 2px 4px rgba(0,0,0,0.05);
    transition: all 0.2s; position: relative; overflow: hidden;
}
.skill-card:active { transform: scale(0.95); }
.skill-card.on-cd { background: #f9f9f9; border-color: #ddd; cursor: not-allowed; color: #aaa; }
.skill-card.condition-fail { background: #fff5f5; border-color: #ffcccc; cursor: not-allowed; opacity: 0.8; }

.ai-skills { background: #f4f4f4; border: none; margin-bottom: 15px; }
.ai-card { border-color: #999; cursor: default; }
.ai-card .s-name { font-size: 12px; margin-bottom: 2px; }

.s-name { font-weight: bold; font-size: 13px; color: #d35400; margin-bottom: 3px; }
.skill-card.on-cd .s-name, .skill-card.condition-fail .s-name { color: #999; }
.s-desc { font-size: 10px; color: #666; line-height: 1.1; margin-bottom: 4px; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; }
.s-info { font-size: 10px; font-weight: bold; padding: 2px 4px; border-radius: 3px; display: inline-block; }
.s-info.ready { color: green; background: #e0f9e0; }
.s-info.condition { color: #c0392b; background: #fadbd8; }

/* 棋盘 */
.board-wrapper { display: flex; justify-content: center; margin-bottom: 15px; }
.board { display: flex; flex-direction: column; border: 2px solid #8b4513; background: #deb887; box-shadow: 0 5px 15px rgba(0,0,0,0.3); }
.board-row { display: flex; }
.cell {
    width: 28px; height: 28px;
    border-right: 1px solid rgba(0,0,0,0.3); border-bottom: 1px solid rgba(0,0,0,0.3);
    display: flex; justify-content: center; align-items: center; cursor: pointer;
}
.board-row:last-child .cell { border-bottom: none; }
.cell:last-child { border-right: none; }

.piece { width: 85%; height: 85%; border-radius: 50%; }
.piece.black { background: radial-gradient(circle at 30% 30%, #666, #000); box-shadow: 1px 1px 2px rgba(0,0,0,0.5); }
.piece.white { background: radial-gradient(circle at 30% 30%, #fff, #ddd); box-shadow: 1px 1px 2px rgba(0,0,0,0.3); }

/* 按钮通用 */
.close-btn { width: 100%; background: #2ecc71; color: white; border: none; padding: 10px; cursor: pointer; border-radius: 5px; font-weight: bold; margin-top: 15px; }
.btn-group button { display: block; width: 100%; margin: 8px 0; padding: 10px; cursor: pointer; border: 1px solid #ccc; background: #f9f9f9; border-radius: 5px; }
.primary-btn { background: #e67e22 !important; color: white !important; border: none !important; font-weight: bold; }
.secondary-btn { background: white !important; color: #666 !important; }
.restart-btn { margin-top: 5px; padding: 10px 25px; background: #7f8c8d; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 14px; width: 100%; }
</style>