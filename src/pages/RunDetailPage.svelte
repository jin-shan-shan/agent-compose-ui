<script lang="ts">
  import { createEventDispatcher, onMount } from 'svelte';

  import { getAutomationRun, listAutomationEvents, type AutomationEvent } from '../api/loaders';
  import {
    getWorkSessionStatus,
    listWorkSessionCells,
    listWorkSessionEvents,
    resumeWorkSession,
    sendWorkSessionMessageStream,
    type WorkSessionEvent,
    type WorkSessionCell,
  } from '../api/sessions';
  import { automationRunToRun, sessionToRun, type ProductRun } from '../model/runs';
  import { formatBeijingTime } from '../time';
  import { CellType } from '@chaitin-ai/agent-compose-client/agentcompose/v1/agentcompose_pb.js';

  export let runId = '';
  export let loaderId = '';
  export let runType = '';

  const dispatch = createEventDispatcher<{ navigateRuns: string }>();

  type LogLine = {
    id: string;
    timestamp: string;
    kind: 'loader_event' | 'session_event' | 'cell_user' | 'cell_agent' | 'cell_system' | 'result' | 'payload';
    level?: string;
    source: string;
    message: string;
    detail?: string;
  };

  let loading = true;
  let error = '';
  let run: ProductRun | null = null;
  let logLines: LogLine[] = [];
  let chatSessionId = '';
  let sessionStatus = '';
  let messageDraft = '';
  let sendingMessage = false;
  let messageAbort: AbortController | null = null;
  let resuming = false;

  onMount(() => { void load(); });

  function pushLines(lines: LogLine[], items: LogLine[]): void {
    for (const item of items) lines.push(item);
  }

  function sortLines(lines: LogLine[]): void {
    lines.sort((a, b) => new Date(a.timestamp).getTime() - new Date(b.timestamp).getTime());
  }

  async function load(): Promise<void> {
    if (!runId) return;
    loading = true;
    error = '';
    try {
      if (runType === 'work_session' || (!runType && !loaderId)) {
        await loadWorkSession();
      } else {
        await loadAutomationRun();
      }
    } catch (err) {
      error = err instanceof Error ? err.message : String(err);
    } finally {
      loading = false;
    }
  }

  async function loadWorkSession(): Promise<void> {
    const [status, cells, events] = await Promise.all([
      getWorkSessionStatus(runId).catch(() => null),
      listWorkSessionCells(runId).catch(() => []),
      listWorkSessionEvents(runId).catch(() => []),
    ]);
    const sessionRun = status ? sessionToRun(status) : null;
    run = {
      id: runId, title: runId, type: 'work_session',
      status: sessionRun?.status || '未知',
      agentId: sessionRun?.agentId || '', agent: sessionRun?.agent || '',
      automationId: sessionRun?.automationId || '', automation: sessionRun?.automation || '',
      sourceSessionTags: status?.tags?.map((t: { name: string; value: string }) => ({ name: t.name, value: t.value })) || [],
      trigger: sessionRun?.trigger || '-', capabilitySet: '',
      workspace: status?.workspacePath || '',
      startedAt: status?.createdAt || '', completedAt: status?.updatedAt || '',
      duration: '-', rawStatus: status?.status || '', agentProvider: '',
      messageCount: cells.length, eventCount: events.length,
      errorSummary: '', output: '', input: '',
      messages: cells.flatMap(cellToMessages),
      events: events.map((e: WorkSessionEvent) => ({ type: e.type, level: e.level, message: e.message, createdAt: e.createdAt })),
      artifacts: [],
    };
    loaderId = run.automationId;
    chatSessionId = runId;
    sessionStatus = run.status;

    const lines: LogLine[] = [];
    for (const cell of cells) {
      lines.push({
        id: cell.id || `cell-${cell.createdAt}`,
        timestamp: cell.createdAt || '',
        kind: cellTypeKind(cell.type),
        source: cellLogSource(cell),
        message: cellMessage(cell),
        detail: cellDetail(cell),
      });
    }
    for (const e of events) {
      lines.push({ id: e.id, timestamp: e.createdAt, kind: 'session_event', level: e.level, source: e.type, message: e.message });
    }
    sortLines(lines);

    if (loaderId) {
      const loaderEvents = await listAutomationEvents(loaderId, 500).catch(() => []);
      for (const le of loaderEvents.filter((e: AutomationEvent) => !e.runId || e.runId === runId)) {
        lines.push({ id: le.id, timestamp: le.createdAt, kind: 'loader_event', level: le.level, source: le.type, message: le.message, detail: le.payloadJson ? tryFormatJson(le.payloadJson) : '' });
      }
      sortLines(lines);
    }
    logLines = lines;
  }

  async function loadAutomationRun(): Promise<void> {
    const [detail, events] = await Promise.all([
      getAutomationRun(loaderId, runId).catch(() => null),
      listAutomationEvents(loaderId, 500).catch(() => []),
    ]);
    if (!detail) { error = '自动化运行记录不存在'; return; }
    const productRun = automationRunToRun(detail);
    run = { ...productRun, events: events.filter((e) => !e.runId || e.runId === runId).map((e: AutomationEvent) => ({ type: e.type, level: e.level, message: e.message, createdAt: e.createdAt })) };

    const lines: LogLine[] = [];
    for (const e of events.filter((e: AutomationEvent) => !e.runId || e.runId === runId)) {
      lines.push({ id: e.id, timestamp: e.createdAt, kind: 'loader_event', level: e.level, source: e.type, message: e.message, detail: e.payloadJson || '' });
    }
    if (detail.resultJson) {
      lines.push({ id: 'result', timestamp: detail.completedAt || detail.startedAt, kind: 'result', source: '执行结果', message: tryFormatJson(detail.resultJson) });
    }
    if (detail.payloadJson) {
      lines.push({ id: 'payload', timestamp: detail.startedAt, kind: 'payload', source: '输入载荷', message: tryFormatJson(detail.payloadJson) });
    }
    sortLines(lines);

    // Linked session: find sessionId from events belonging to this run
    const sessionCandidates: Array<{ source: string; id: string }> = [];
    for (const e of events.filter((e: AutomationEvent) => !e.runId || e.runId === runId)) {
      if (e.linkedSessionId) sessionCandidates.push({ source: `linkedSessionId (${e.type})`, id: e.linkedSessionId });
      if (e.linkedAgentSessionId) sessionCandidates.push({ source: `linkedAgentSessionId (${e.type})`, id: e.linkedAgentSessionId });
      if (e.payloadJson) {
        const extracted = tryExtractSession(e.payloadJson);
        if (extracted) sessionCandidates.push({ source: `payloadJson (${e.type})`, id: extracted });
      }
    }

    const linkedSessionId =
      events.find((e: AutomationEvent) => e.linkedSessionId && e.runId === runId)?.linkedSessionId ||
      events.find((e: AutomationEvent) => e.linkedAgentSessionId && e.runId === runId)?.linkedAgentSessionId ||
      events.find((e: AutomationEvent) => e.linkedSessionId && !e.runId)?.linkedSessionId ||
      events.find((e: AutomationEvent) => e.linkedAgentSessionId && !e.runId)?.linkedAgentSessionId ||
      events.find((e: AutomationEvent) => e.linkedSessionId)?.linkedSessionId ||
      events.find((e: AutomationEvent) => e.linkedAgentSessionId)?.linkedAgentSessionId ||
      extractSessionFromPayload(events) ||
      '';
    if (linkedSessionId) {
      chatSessionId = linkedSessionId;
      try {
        const [cells, sessionEvents, sessStatus] = await Promise.all([
          listWorkSessionCells(linkedSessionId).catch(() => []),
          listWorkSessionEvents(linkedSessionId).catch(() => []),
          getWorkSessionStatus(linkedSessionId).catch(() => null),
        ]);
        if (sessStatus) sessionStatus = mapStatus(sessStatus.status);
        for (const cell of cells) {
          lines.push({ id: cell.id || `scell-${cell.createdAt}`, timestamp: cell.createdAt || '', kind: cellTypeKind(cell.type), source: cellLogSource(cell) + ' (会话)', message: cellMessage(cell), detail: cellDetail(cell) });
        }
        for (const se of sessionEvents) {
          lines.push({ id: se.id, timestamp: se.createdAt, kind: 'session_event', level: se.level, source: se.type, message: se.message });
        }
      } catch { /* ignore */ }
    }

    sortLines(lines);
    logLines = lines;
  }

  const AGENT_RESULT_PREFIX = '__AGENT_RESULT__';

  function extractAgentResultText(raw: string): string {
    const index = raw.indexOf(AGENT_RESULT_PREFIX);
    if (index < 0) return raw;
    const jsonStart = raw.indexOf('{', index);
    if (jsonStart < 0) return raw.slice(0, index);
    try {
      const jsonEnd = raw.lastIndexOf('}') + 1;
      const payload = JSON.parse(raw.slice(jsonStart, jsonEnd));
      const text = payload.finalText || payload.transcript || payload.stderr || '';
      const before = raw.slice(0, index).trim();
      return before ? `${before}\n${text}` : text;
    } catch {
      return raw.slice(0, index);
    }
  }

  function cellMessage(cell: WorkSessionCell): string {
    return extractAgentResultText(cell.output || cell.source || '');
  }

  function cellTypeKind(t: CellType | undefined): LogLine['kind'] {
    if (t === CellType.AGENT) return 'cell_agent';
    if (t === CellType.UNSPECIFIED) return 'cell_user';
    return 'cell_system';
  }

  function cellTypeLabel(t: CellType | undefined): string {
    if (t === CellType.AGENT) return '智能体';
    if (t === CellType.UNSPECIFIED) return '用户';
    return '系统';
  }

  function cellLogSource(cell: WorkSessionCell): string {
    if (cell.agent && cell.source) return `${cell.agent} · ${cell.source}`;
    if (cell.agent) return cell.agent;
    return cellTypeLabel(cell.type);
  }

  function cellDetail(cell: WorkSessionCell): string {
    const parts: string[] = [];
    if (cell.stopReason) parts.push(`停止原因: ${cell.stopReason}`);
    if (cell.exitCode !== undefined && cell.exitCode !== 0) parts.push(`exit: ${cell.exitCode}`);
    return parts.join(' | ');
  }

  function cellToMessages(cell: WorkSessionCell): ProductRun['messages'] {
    const role: 'user' | 'agent' | 'system' =
      cell.type === CellType.UNSPECIFIED ? 'user' : cell.type === CellType.AGENT ? 'agent' : 'system';
    return [{ id: cell.id, role, type: cell.type, content: cell.output, at: cell.createdAt || '', agent: cell.agent, source: cell.source, failed: !cell.success && cell.exitCode !== 0, success: cell.exitCode === 0, exitCode: cell.exitCode, stopReason: cell.stopReason }];
  }

  function tryFormatJson(raw: string): string {
    try { return JSON.stringify(JSON.parse(raw), null, 2); } catch { return raw; }
  }

  function extractSessionFromPayload(events: AutomationEvent[]): string {
    for (const e of events) {
      const found = tryExtractSession(e.payloadJson);
      if (found) return found;
    }
    return '';
  }

  function tryExtractSession(payloadJson: string): string {
    if (!payloadJson) return '';
    // Strip __AGENT_RESULT__ prefix if present
    const clean = payloadJson.includes('__AGENT_RESULT__') ? payloadJson.substring(payloadJson.lastIndexOf('__AGENT_RESULT__') + '__AGENT_RESULT__'.length) : payloadJson;
    try {
      const payload = JSON.parse(clean);
      if (payload.sessionId) return String(payload.sessionId);
      if (payload.agentSessionId) return String(payload.agentSessionId);
      if (payload.SessionID) return String(payload.SessionID);
      if (payload.session_id) return String(payload.session_id);
    } catch { /* ignore */ }
    try {
      const payload = JSON.parse(payloadJson);
      if (payload.sessionId) return String(payload.sessionId);
    } catch { /* ignore */ }
    return '';
  }

  function mapStatus(s: string): string {
    const n = (s || '').toUpperCase();
    if (s === '启动中' || n === 'PENDING') return '启动中';
    if (s === '运行中' || n === 'RUNNING') return '运行中';
    if (s === '启动失败' || n === 'FAILED') return '启动失败';
    if (s === '失败') return '失败';
    if (s === '成功' || n === 'SUCCEEDED') return '成功';
    if (s === '已停止' || n === 'STOPPED') return '已停止';
    if (s === '等待中' || n === 'WAITING') return '等待中';
    if (s === '跳过' || n === 'SKIPPED') return '跳过';
    if (s === '已取消' || n === 'CANCELLED') return '已取消';
    return s || '未知';
  }

  function statusToneValue(s: string): 'blue' | 'green' | 'red' | 'gray' {
    const v = mapStatus(s);
    if (['启动失败', '失败', '跳过', '已取消'].includes(v)) return 'red';
    if (['成功', '已停止'].includes(v)) return 'green';
    if (['启动中', '等待中', '运行中', '恢复中', '停止中'].includes(v)) return 'blue';
    return 'gray';
  }

  function logLevelTone(level: string | undefined): string {
    const v = (level || '').toLowerCase();
    if (v === 'error' || v === 'critical' || v === 'fatal') return 'error';
    if (v === 'warning' || v === 'warn') return 'warning';
    return 'default';
  }

  function translateSource(source: string): string {
    const map: Record<string, string> = {
      'loader.run.skipped': 'Run 跳过', 'loader.run.started': 'Run 开始', 'loader.run.completed': 'Run 完成', 'loader.run.failed': 'Run 失败',
      'loader.log': '自定义日志', 'loader.event.published': '事件发布', 'loader.event.publish.failed': '事件发布失败',
      'loader.session.resumed': 'Session 就绪', 'loader.session.rpc.completed': 'RPC 完成', 'loader.session.rpc.failed': 'RPC 失败',
      'loader.agent.completed': 'Agent 完成', 'loader.agent.failed': 'Agent 失败',
      'loader.session.stopped': 'Session 停止', 'loader.session.stop_failed': 'Session 停止失败',
      'loader.command.completed': '命令完成', 'loader.command.failed': '命令失败',
      'loader.llm.completed': 'LLM 完成', 'loader.llm.failed': 'LLM 失败',
      'session.created': 'Session 创建', 'session.resumed': 'Session 恢复', 'session.runtime_lost': 'Runtime 失联', 'session.stopped': 'Session 停止',
      'agent.user': '用户消息', 'agent.assistant': 'Agent 回复', 'agent.assistant.failed': 'Agent 失败',
    };
    return map[source] || source;
  }

  function kindBadge(kind: LogLine['kind']): string {
    if (kind === 'loader_event') return '⛭';
    if (kind === 'session_event') return '○';
    if (kind === 'cell_user') return '▶';
    if (kind === 'cell_agent') return '◀';
    if (kind === 'cell_system') return '◆';
    if (kind === 'result') return '✓';
    if (kind === 'payload') return '⬇';
    return '·';
  }

  function kindLabel(kind: LogLine['kind']): string {
    if (kind === 'loader_event') return '任务事件';
    if (kind === 'session_event') return '会话事件';
    if (kind === 'cell_user') return '用户输入';
    if (kind === 'cell_agent') return 'Agent 回复';
    if (kind === 'cell_system') return '系统';
    if (kind === 'result') return '执行结果';
    if (kind === 'payload') return '输入载荷';
    return '';
  }

  function kindBorderColor(kind: LogLine['kind']): string {
    if (kind === 'cell_user' || kind === 'payload') return 'var(--blue,#3b82f6)';
    if (kind === 'cell_agent' || kind === 'result') return 'var(--green,#22c55e)';
    if (kind === 'loader_event') return 'var(--amber,#f59e0b)';
    if (kind === 'session_event') return 'var(--slate,#94a3b8)';
    if (kind === 'cell_system') return 'var(--muted,#64748b)';
    return 'var(--muted,#64748b)';
  }

  function formatTime(value: string | undefined): string {
    return value ? formatBeijingTime(value) : '-';
  }

  function shortId(id: string): string { return id ? id.substring(0, 8) : ''; }

  function tagValue(tags: Array<{ name: string; value: string }> | undefined, name: string): string {
    return tags?.find((t) => t.name === name)?.value || '-';
  }

  function sourceText(r: ProductRun): string {
    if (r.type === 'work_session') return r.automationId ? '自动化任务 (工作会话)' : '手动对话';
    return '自动化运行';
  }

  function endedAt(r: ProductRun): string { return r.completedAt || '-'; }

  function durationText(r: ProductRun): string {
    if (r.duration && r.duration !== '-') return r.duration;
    if (!r.startedAt) return '-';
    const s = Math.round(((r.completedAt ? new Date(r.completedAt).getTime() : Date.now()) - new Date(r.startedAt).getTime()) / 1000);
    if (s < 60) return `${s}s`;
    if (s < 3600) return `${Math.floor(s / 60)}m ${s % 60}s`;
    return `${Math.floor(s / 3600)}h ${Math.floor((s % 3600) / 60)}m`;
  }

  function latestAgentName(): string {
    const messages = run?.messages || [];
    for (let i = messages.length - 1; i >= 0; i--) {
      if (messages[i].agent) return messages[i].agent!;
    }
    return 'codex';
  }

  function canResume(): boolean {
    return Boolean(chatSessionId) && ['已停止', '启动失败', '失败'].includes(sessionStatus) && !resuming;
  }

  async function resumeSession(): Promise<void> {
    if (!canResume() || !chatSessionId) return;
    resuming = true;
    error = '';
    try {
      await resumeWorkSession(chatSessionId);
      await load();
    } catch (err) {
      error = err instanceof Error ? err.message : String(err);
    } finally {
      resuming = false;
    }
  }

  function canSendMessage(): boolean {
    return Boolean(chatSessionId) && sessionStatus === '运行中' && !sendingMessage;
  }

  function messageInputHint(): string {
    if (!chatSessionId) return '未关联工作会话';
    if (sendingMessage) return '正在回复...';
    if (sessionStatus === '运行中') return 'Enter 发送，Shift + Enter 换行';
    return `会话${sessionStatus || '未运行'}`;
  }

  async function sendMessage(): Promise<void> {
    const message = messageDraft.trim();
    if (!message || !canSendMessage() || !chatSessionId) return;
    messageAbort?.abort();
    const controller = new AbortController();
    messageAbort = controller;
    sendingMessage = true;
    messageDraft = '';
    error = '';
    const sentAt = Date.now();
    const userCellId = `pending-user-${sentAt}`;
    let agentCellId = `pending-agent-${sentAt}`;
    const agent = latestAgentName();
    logLines = [...logLines,
      { id: userCellId, timestamp: new Date(sentAt).toISOString(), kind: 'cell_user', source: '用户', message },
      { id: agentCellId, timestamp: new Date(sentAt).toISOString(), kind: 'cell_agent', source: agent, message: '' },
    ];
    try {
      await sendWorkSessionMessageStream(chatSessionId, agent, message, (event) => {
        if (event.type === 'started' && event.runId) {
          logLines = logLines.map(l => l.id === agentCellId ? { ...l, id: event.runId } : l);
          agentCellId = event.runId;
        } else if (event.type === 'chunk' && event.chunk) {
          logLines = logLines.map(l => l.id === agentCellId ? { ...l, message: l.message + event.chunk } : l);
        } else if (event.type === 'completed' && event.run) {
          const run = event.run;
          logLines = logLines.map(l => l.id === (run.id || agentCellId)
            ? { ...l, message: run.output || l.message, detail: run.stopReason || '' }
            : l);
        }
      }, controller.signal);
      sendingMessage = false;
      await load();
    } catch (err) {
      if (!controller.signal.aborted) {
        logLines = logLines.map(l => l.id === agentCellId ? { ...l, detail: `发送失败: ${err instanceof Error ? err.message : String(err)}` } : l);
        error = err instanceof Error ? err.message : String(err);
      }
    } finally {
      sendingMessage = false;
      if (messageAbort === controller) messageAbort = null;
    }
  }

  function handleMessageKeydown(event: KeyboardEvent): void {
    if (event.key !== 'Enter' || event.shiftKey || event.metaKey || event.ctrlKey) return;
    event.preventDefault();
    void sendMessage();
  }

  $: statusLabel = mapStatus(run?.status || '');
  $: statusTone = statusToneValue(statusLabel);
</script>

{#if loading}
  <div class="alert info">正在加载运行详情...</div>
{:else if error}
  <div class="alert danger">{error}</div>
{:else if !run}
  <div class="alert info">运行记录不存在</div>
{:else}

<div class="page-title">
  <div><h2>运行详情</h2></div>
  <div class="toolbar">
    <button on:click={() => dispatch('navigateRuns', runId)}>返回运行中心</button>
    <button
      disabled={!canResume()}
      on:click={resumeSession}
    >
      {resuming ? '重启中...' : '重启 Session'}
    </button>
    <button on:click={load}>{loading ? '加载中...' : '刷新'}</button>
  </div>
</div>

<div class="debug-workbench">
  <section class="debug-card debug-terminal-panel">
    <div class="debug-card-head">
      <h3>
        {run.type === 'work_session' ? '工作会话' : '自动化运行'}
        · #{shortId(run.id)}
        <em class={`home-pill ${statusTone}`} style="margin-left: 8px;">{statusLabel}</em>
        <span style="margin-left: 12px; font-weight: 400; color: var(--muted); font-size: var(--font-size-sm);">{logLines.length} 条日志</span>
      </h3>
    </div>

    {#if logLines.length === 0}
      <div class="empty compact-empty" style="padding: 60px 0;">暂无日志记录。</div>
    {:else}
      <div class="run-detail-log">
        {#each logLines as line, index (`${line.id}-${index}`)}
          <div
            class="log-row log-kind-{line.kind} log-level-{logLevelTone(line.level)}"
            class:log-row-alt={index % 2 === 1}
            style="--log-border: {kindBorderColor(line.kind)}"
          >
            <span class="log-gutter">
              <span class="log-num">{index + 1}</span>
              <span class="log-badge" title={kindLabel(line.kind)}>{kindBadge(line.kind)}</span>
            </span>
            <span class="log-content">
              <span class="log-meta">
                <span class="log-time">{formatTime(line.timestamp)}</span>
                <span class="log-kind-label">{kindLabel(line.kind)}</span>
                <span class="log-source">[{translateSource(line.source)}]</span>
                {#if line.level}
                  <span class="log-level log-level-pill-{logLevelTone(line.level)}">{line.level}</span>
                {/if}
              </span>
              <span class="log-body">{line.message}</span>
              {#if line.detail}
                <span class="log-detail">{line.detail}</span>
              {/if}
            </span>
          </div>
        {/each}
      </div>
    {/if}

    <div class="message-composer" class:disabled={!canSendMessage()}>
      <textarea
        rows="3"
        value={messageDraft}
        placeholder={messageInputHint()}
        disabled={!canSendMessage()}
        on:input={(event) => messageDraft = event.currentTarget.value}
        on:keydown={(event) => handleMessageKeydown(event)}
      ></textarea>
      <div class="composer-actions">
        <span>{messageInputHint()}</span>
        <button
          class="compact-button primary-action"
          disabled={!canSendMessage() || !messageDraft.trim()}
          on:click={() => sendMessage()}
        >
          {sendingMessage ? '发送中...' : '发送'}
        </button>
      </div>
    </div>
  </section>

  <aside class="debug-side-panel">
    <section class="debug-card debug-side-card">
      <div class="debug-card-head"><h3>运行信息</h3></div>
      <div class="descriptions-small debug-descriptions">
        <div><span>运行 ID</span><b><span class="mono-text">{runId}</span></b></div>
        <div><span>运行类型</span><b>{sourceText(run)}</b></div>
        <div><span>运行状态</span><b><em class={`home-pill ${statusTone}`}>{statusLabel}</em></b></div>
        <div><span>触发方式</span><b>{run.trigger || '-'}</b></div>
        <div><span>开始时间</span><b>{formatTime(run.startedAt)}</b></div>
        <div><span>结束时间</span><b>{formatTime(endedAt(run))}</b></div>
        <div><span>运行耗时</span><b>{durationText(run)}</b></div>
        <div><span>日志总数</span><b>{logLines.length} 条</b></div>
      </div>
    </section>

    <section class="debug-card debug-side-card">
      <div class="debug-card-head"><h3>任务信息</h3></div>
      <div class="descriptions-small debug-descriptions">
        <div><span>自动化任务</span><b>{#if run.automationId}<span class="mono-text">{run.automationId}</span>{:else}<span class="muted">-</span>{/if}</b></div>
        <div><span>任务名称</span><b>{run.automation || '-'}</b></div>
        <div><span>关联智能体</span><b>{run.agent || tagValue(run.sourceSessionTags, 'agent_id') || '-'}</b></div>
        {#if run.errorSummary}
          <div><span>错误摘要</span><b class="danger-text">{run.errorSummary}</b></div>
        {/if}
      </div>
    </section>

    {#if run.type === 'work_session'}
      <section class="debug-card debug-side-card">
        <div class="debug-card-head"><h3>会话信息</h3></div>
        <div class="descriptions-small debug-descriptions">
          <div><span>工作空间</span><b>{#if run.workspace}<span class="mono-text">{run.workspace}</span>{:else}<span class="muted">-</span>{/if}</b></div>
          <div><span>原始状态</span><b><span class="mono-text">{run.rawStatus}</span></b></div>
          <div><span>消息/事件</span><b>{run.messageCount} / {run.eventCount}</b></div>
        </div>
      </section>
    {/if}


    {#if run.sourceSessionTags && run.sourceSessionTags.length > 0}
      <section class="debug-card debug-side-card">
        <div class="debug-card-head"><h3>上下文标签</h3></div>
        <div class="descriptions-small debug-descriptions">
          {#each run.sourceSessionTags as tag}
            <div><span>{tag.name}</span><b><span class="mono-text">{tag.value || '-'}</span></b></div>
          {/each}
        </div>
      </section>
    {/if}
  </aside>
</div>
{/if}

<style>
  .debug-workbench :global(.debug-terminal-panel) {
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }
  .run-detail-log {
    height: 700px;
    overflow-y: auto;
    margin: 0;
    padding: 8px 0;
    border: 1px solid rgba(255,255,255,0.08);
    border-radius: 6px;
    background: #07111a;
    color: #d8e2ec;
    font-family: var(--mono);
    font-size: 13px;
    line-height: 1.5;
  }
  .log-row {
    display: flex;
    gap: 0;
    border-left: 3px solid var(--log-border, transparent);
    padding: 6px 10px 6px 8px;
    min-width: 0;
  }
  .log-row-alt {
    background: rgba(255,255,255,0.015);
  }
  .log-row.log-level-error {
    background: rgba(239,68,68,0.06);
  }
  .log-row.log-level-warning {
    background: rgba(245,158,11,0.04);
  }

  .log-gutter {
    flex: 0 0 auto;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 2px;
    width: 36px;
    padding-right: 10px;
    opacity: 0.7;
  }
  .log-num {
    font-size: 10px;
    color: #4a5f73;
    user-select: none;
    line-height: 1;
  }
  .log-badge {
    font-size: 14px;
    line-height: 1;
    user-select: none;
  }

  .log-content {
    flex: 1 1 auto;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 1px;
  }

  .log-meta {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
  }
  .log-time {
    color: #4a5f73;
    font-size: 11px;
    white-space: nowrap;
  }
  .log-kind-label {
    font-size: 10px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.4px;
    white-space: nowrap;
    padding: 1px 5px;
    border-radius: 3px;
  }
  .log-kind-cell_user .log-kind-label { background: rgba(59,130,246,0.2); color: #60a5fa; }
  .log-kind-cell_agent .log-kind-label { background: rgba(34,197,94,0.2); color: #4ade80; }
  .log-kind-cell_system .log-kind-label { background: rgba(148,163,184,0.15); color: #94a3b8; }
  .log-kind-loader_event .log-kind-label { background: rgba(245,158,11,0.18); color: #fbbf24; }
  .log-kind-session_event .log-kind-label { background: rgba(148,163,184,0.12); color: #cbd5e1; }
  .log-kind-result .log-kind-label { background: rgba(34,197,94,0.2); color: #4ade80; }
  .log-kind-payload .log-kind-label { background: rgba(59,130,246,0.18); color: #60a5fa; }

  .log-source {
    color: #7c8fa0;
    font-size: 11px;
    white-space: nowrap;
  }
  .log-level {
    font-size: 10px;
    padding: 1px 5px;
    border-radius: 3px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.3px;
  }
  .log-level-pill-error { background: rgba(239,68,68,0.25); color: #f87171; }
  .log-level-pill-warning { background: rgba(245,158,11,0.25); color: #fbbf24; }
  .log-level-pill-default { background: rgba(255,255,255,0.06); color: #6b7f94; }

  .log-body {
    color: #d8e2ec;
    word-break: break-word;
    white-space: pre-wrap;
  }
  .log-kind-cell_user .log-body { color: #93c5fd; }
  .log-kind-cell_agent .log-body { color: #86efac; }
  .log-kind-result .log-body { color: #86efac; white-space: pre-wrap; }
  .log-kind-payload .log-body { color: #93c5fd; white-space: pre-wrap; }
  .log-level-error .log-body { color: #fca5a5; }

  .log-detail {
    margin-top: 4px;
    padding: 6px 10px;
    background: rgba(255,255,255,0.03);
    border-left: 2px solid rgba(255,255,255,0.1);
    border-radius: 0 4px 4px 0;
    color: #7c8fa0;
    font-size: 12px;
    white-space: pre-wrap;
    word-break: break-word;
  }

  .message-composer {
    flex: 0 0 auto;
    display: grid;
    gap: 7px;
    margin-top: 10px;
    padding: 10px;
    border: 1px solid var(--line);
    border-radius: 8px;
    background: #fff;
  }
  .message-composer.disabled {
    background: var(--surface-2);
  }
  .message-composer textarea {
    width: 100%;
    min-height: 64px;
    resize: none;
    border: 1px solid var(--line);
    border-radius: 6px;
    padding: 9px 10px;
    color: var(--text);
    background: #fff;
    font: inherit;
    line-height: 1.45;
    box-sizing: border-box;
  }
  .message-composer textarea:disabled {
    color: var(--muted);
    background: var(--surface-2);
  }
  .composer-actions {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
  }
  .composer-actions span {
    min-width: 0;
    color: var(--muted);
    font-size: 12px;
  }
  .compact-button {
    flex: 0 0 auto;
    min-height: 28px;
    padding: 5px 12px;
    font-size: 12px;
    font-weight: 600;
    border-radius: 5px;
    border: 1px solid var(--line);
    background: #fff;
    color: var(--text);
    cursor: pointer;
  }
  .compact-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
  .primary-action {
    border-color: var(--primary);
    background: var(--primary);
    color: #fff;
  }
</style>
