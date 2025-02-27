<template>
  <div class="ui container action-view-container">
    <div class="action-view-header">
      <div class="action-info-summary">
        <div class="action-info-summary-title">
          <ActionRunStatus :locale-status="locale.status[run.status]" :status="run.status" :size="20"/>
          <h2 class="action-info-summary-title-text">
            {{ run.title }}
          </h2>
        </div>
        <button class="ui basic small compact button primary" @click="approveRun()" v-if="run.canApprove">
          {{ locale.approve }}
        </button>
        <button class="ui basic small compact button red" @click="cancelRun()" v-else-if="run.canCancel">
          {{ locale.cancel }}
        </button>
        <button class="ui basic small compact button secondary" @click="rerun()" v-else-if="run.canRerun">
          {{ locale.rerun_all }}
        </button>
      </div>
      <div class="action-commit-summary">
        {{ run.commit.localeCommit }}
        <a :href="run.commit.link">{{ run.commit.shortSHA }}</a>
        <span class="ui label" v-if="run.commit.shortSHA">
          <a :href="run.commit.branch.link">{{ run.commit.branch.name }}</a>
        </span>
        {{ run.commit.localePushedBy }}
        <a :href="run.commit.pusher.link">{{ run.commit.pusher.displayName }}</a>
      </div>
    </div>
    <div class="action-view-body">
      <div class="action-view-left">
        <div class="job-group-section">
          <div class="job-brief-list">
            <div class="job-brief-item" :class="parseInt(jobIndex) === index ? 'selected' : ''" v-for="(job, index) in run.jobs" :key="job.id" @mouseenter="onHoverRerunIndex = job.id" @mouseleave="onHoverRerunIndex = -1">
              <a class="job-brief-link" :href="run.link+'/jobs/'+index">
                <ActionRunStatus :locale-status="locale.status[job.status]" :status="job.status"/>
                <span class="job-brief-name gt-mx-3 gt-ellipsis">{{ job.name }}</span>
              </a>
              <span class="job-brief-info">
                <span class="step-summary-duration">{{ job.duration }}</span>
                <SvgIcon name="octicon-sync" role="button" :data-tooltip-content="locale.rerun" class="job-brief-rerun gt-mx-3" @click="rerunJob(index)" v-if="job.canRerun && onHoverRerunIndex === job.id"/>
              </span>
            </div>
          </div>
        </div>
        <div class="job-artifacts" v-if="artifacts.length > 0">
          <div class="job-artifacts-title">
            {{ locale.artifactsTitle }}
          </div>
          <ul class="job-artifacts-list">
            <li class="job-artifacts-item" v-for="artifact in artifacts" :key="artifact.id">
              <a class="job-artifacts-link" target="_blank" :href="run.link+'/artifacts/'+artifact.id">
                <SvgIcon name="octicon-file" class="ui text black job-artifacts-icon" />{{ artifact.name }}
              </a>
            </li>
          </ul>
        </div>
      </div>

      <div class="action-view-right">
        <div class="job-info-header">
          <h3 class="job-info-header-title">
            {{ currentJob.title }}
          </h3>
          <p class="job-info-header-detail">
            {{ currentJob.detail }}
          </p>
        </div>
        <div class="job-step-container">
          <div class="job-step-section" v-for="(jobStep, i) in currentJob.steps" :key="i">
            <div class="job-step-summary" @click.stop="toggleStepLogs(i)" :class="currentJobStepsStates[i].expanded ? 'selected' : ''">
              <!-- If the job is done and the job step log is loaded for the first time, show the loading icon
                currentJobStepsStates[i].cursor === null means the log is loaded for the first time
              -->
              <SvgIcon v-if="isDone(run.status) && currentJobStepsStates[i].expanded && currentJobStepsStates[i].cursor === null" name="octicon-sync" class="gt-mr-3 job-status-rotate"/>
              <SvgIcon v-else :name="currentJobStepsStates[i].expanded ? 'octicon-chevron-down': 'octicon-chevron-right'" class="gt-mr-3"/>
              <ActionRunStatus :status="jobStep.status" class="gt-mr-3"/>

              <span class="step-summary-msg gt-ellipsis">{{ jobStep.summary }}</span>
              <span class="step-summary-duration">{{ jobStep.duration }}</span>
            </div>

            <!-- the log elements could be a lot, do not use v-if to destroy/reconstruct the DOM -->
            <div class="job-step-logs" ref="logs" v-show="currentJobStepsStates[i].expanded"/>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import {SvgIcon} from '../svg.js';
import ActionRunStatus from './ActionRunStatus.vue';
import {createApp} from 'vue';
import AnsiToHTML from 'ansi-to-html';

const {csrfToken} = window.config;

const ansiLogRender = new AnsiToHTML({escapeXML: true});

const sfc = {
  name: 'RepoActionView',
  components: {
    SvgIcon,
    ActionRunStatus,
  },
  props: {
    runIndex: String,
    jobIndex: String,
    actionsURL: String,
    locale: Object,
  },

  data() {
    return {
      // internal state
      loading: false,
      intervalID: null,
      currentJobStepsStates: [],
      artifacts: [],
      onHoverRerunIndex: -1,

      // provided by backend
      run: {
        link: '',
        title: '',
        status: '',
        canCancel: false,
        canApprove: false,
        canRerun: false,
        done: false,
        jobs: [
          // {
          //   id: 0,
          //   name: '',
          //   status: '',
          //   canRerun: false,
          //   duration: '',
          // },
        ],
        commit: {
          localeCommit: '',
          localePushedBy: '',
          shortSHA: '',
          link: '',
          pusher: {
            displayName: '',
            link: '',
          },
          branch: {
            name: '',
            link: '',
          },
        }
      },
      currentJob: {
        title: '',
        detail: '',
        steps: [
          // {
          //   summary: '',
          //   duration: '',
          //   status: '',
          // }
        ],
      },
    };
  },

  mounted() {
    // load job data and then auto-reload periodically
    this.loadJob();
    this.intervalID = setInterval(this.loadJob, 1000);
  },

  unmounted() {
    // clear the interval timer when the component is unmounted
    // even our page is rendered once, not spa style
    if (this.intervalID) {
      clearInterval(this.intervalID);
      this.intervalID = null;
    }
  },

  methods: {
    // get the active container element, either the `job-step-logs` or the `job-log-list` in the `job-log-group`
    getLogsContainer(idx) {
      const el = this.$refs.logs[idx];
      return el._stepLogsActiveContainer ?? el;
    },
    // begin a log group
    beginLogGroup(idx) {
      const el = this.$refs.logs[idx];

      const elJobLogGroup = document.createElement('div');
      elJobLogGroup.classList.add('job-log-group');

      const elJobLogGroupSummary = document.createElement('div');
      elJobLogGroupSummary.classList.add('job-log-group-summary');

      const elJobLogList = document.createElement('div');
      elJobLogList.classList.add('job-log-list');

      elJobLogGroup.append(elJobLogGroupSummary);
      elJobLogGroup.append(elJobLogList);
      el._stepLogsActiveContainer = elJobLogList;
    },
    // end a log group
    endLogGroup(idx) {
      const el = this.$refs.logs[idx];
      el._stepLogsActiveContainer = null;
    },

    // show/hide the step logs for a step
    toggleStepLogs(idx) {
      this.currentJobStepsStates[idx].expanded = !this.currentJobStepsStates[idx].expanded;
      if (this.currentJobStepsStates[idx].expanded) {
        this.loadJob(); // try to load the data immediately instead of waiting for next timer interval
      }
    },
    // rerun a job
    async rerunJob(idx) {
      const jobLink = `${this.run.link}/jobs/${idx}`;
      await this.fetchPost(`${jobLink}/rerun`);
      window.location.href = jobLink;
    },
    // rerun workflow
    async rerun() {
      await this.fetchPost(`${this.run.link}/rerun`);
      window.location.href = this.run.link;
    },
    // cancel a run
    cancelRun() {
      this.fetchPost(`${this.run.link}/cancel`);
    },
    // approve a run
    approveRun() {
      this.fetchPost(`${this.run.link}/approve`);
    },

    createLogLine(line) {
      const div = document.createElement('div');
      div.classList.add('job-log-line');
      div._jobLogTime = line.timestamp;

      const lineNumber = document.createElement('div');
      lineNumber.className = 'line-num';
      lineNumber.textContent = line.index;
      div.append(lineNumber);

      // TODO: Support displaying time optionally

      const logMessage = document.createElement('div');
      logMessage.className = 'log-msg';
      logMessage.innerHTML = ansiLogToHTML(line.message);
      div.append(logMessage);

      return div;
    },

    appendLogs(stepIndex, logLines) {
      for (const line of logLines) {
        // TODO: group support: ##[group]GroupTitle , ##[endgroup]
        const el = this.getLogsContainer(stepIndex);
        el.append(this.createLogLine(line));
      }
    },

    async fetchJob() {
      const logCursors = this.currentJobStepsStates.map((it, idx) => {
        // cursor is used to indicate the last position of the logs
        // it's only used by backend, frontend just reads it and passes it back, it and can be any type.
        // for example: make cursor=null means the first time to fetch logs, cursor=eof means no more logs, etc
        return {step: idx, cursor: it.cursor, expanded: it.expanded};
      });
      const resp = await this.fetchPost(
        `${this.actionsURL}/runs/${this.runIndex}/jobs/${this.jobIndex}`,
        JSON.stringify({logCursors}),
      );
      return await resp.json();
    },

    async loadJob() {
      if (this.loading) return;
      try {
        this.loading = true;

        // refresh artifacts if upload-artifact step done
        const resp = await this.fetchPost(`${this.actionsURL}/runs/${this.runIndex}/artifacts`);
        const artifacts = await resp.json();
        this.artifacts = artifacts['artifacts'] || [];

        const response = await this.fetchJob();

        // save the state to Vue data, then the UI will be updated
        this.run = response.state.run;
        this.currentJob = response.state.currentJob;

        // sync the currentJobStepsStates to store the job step states
        for (let i = 0; i < this.currentJob.steps.length; i++) {
          if (!this.currentJobStepsStates[i]) {
            // initial states for job steps
            this.currentJobStepsStates[i] = {cursor: null, expanded: false};
          }
        }
        // append logs to the UI
        for (const logs of response.logs.stepsLog) {
          // save the cursor, it will be passed to backend next time
          this.currentJobStepsStates[logs.step].cursor = logs.cursor;
          this.appendLogs(logs.step, logs.lines);
        }

        if (this.run.done && this.intervalID) {
          clearInterval(this.intervalID);
          this.intervalID = null;
        }
      } finally {
        this.loading = false;
      }
    },


    fetchPost(url, body) {
      return fetch(url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Csrf-Token': csrfToken,
        },
        body,
      });
    },

    isDone(status) {
      return ['success', 'skipped', 'failure', 'cancelled'].includes(status);
    }
  },
};

export default sfc;

export function initRepositoryActionView() {
  const el = document.getElementById('repo-action-view');
  if (!el) return;

  // TODO: the parent element's full height doesn't work well now,
  // but we can not pollute the global style at the moment, only fix the height problem for pages with this component
  const parentFullHeight = document.querySelector('body > div.full.height');
  if (parentFullHeight) parentFullHeight.style.paddingBottom = '0';

  const view = createApp(sfc, {
    runIndex: el.getAttribute('data-run-index'),
    jobIndex: el.getAttribute('data-job-index'),
    actionsURL: el.getAttribute('data-actions-url'),
    locale: {
      approve: el.getAttribute('data-locale-approve'),
      cancel: el.getAttribute('data-locale-cancel'),
      rerun: el.getAttribute('data-locale-rerun'),
      artifactsTitle: el.getAttribute('data-locale-artifacts-title'),
      rerun_all: el.getAttribute('data-locale-rerun-all'),
      status: {
        unknown: el.getAttribute('data-locale-status-unknown'),
        waiting: el.getAttribute('data-locale-status-waiting'),
        running: el.getAttribute('data-locale-status-running'),
        success: el.getAttribute('data-locale-status-success'),
        failure: el.getAttribute('data-locale-status-failure'),
        cancelled: el.getAttribute('data-locale-status-cancelled'),
        skipped: el.getAttribute('data-locale-status-skipped'),
        blocked: el.getAttribute('data-locale-status-blocked'),
      }
    }
  });
  view.mount(el);
}

// some unhandled control sequences by AnsiToHTML
// https://man7.org/linux/man-pages/man4/console_codes.4.html
const ansiRegexpRemove = /\x1b\[\d+[A-H]/g; // Move cursor, treat them as no-op.
const ansiRegexpNewLine = /\x1b\[\d?[JK]/g; // Erase display/line, treat them as a Carriage Return

function ansiCleanControlSequences(line) {
  if (line.includes('\x1b')) {
    line = line.replace(ansiRegexpRemove, '');
    line = line.replace(ansiRegexpNewLine, '\r');
  }
  return line;
}

export function ansiLogToHTML(line) {
  if (line.endsWith('\r\n')) {
    line = line.substring(0, line.length - 2);
  } else if (line.endsWith('\n')) {
    line = line.substring(0, line.length - 1);
  }

  // usually we do not need to process control chars like "\033[", let AnsiToHTML do it
  // but AnsiToHTML has bugs, so we need to clean some control sequences first
  line = ansiCleanControlSequences(line);

  if (!line.includes('\r')) {
    return ansiLogRender.toHtml(line);
  }

  // handle "\rReading...1%\rReading...5%\rReading...100%",
  // convert it into a multiple-line string: "Reading...1%\nReading...5%\nReading...100%"
  const lines = [];
  for (const part of line.split('\r')) {
    if (part === '') continue;
    const partHtml = ansiLogRender.toHtml(part);
    if (partHtml !== '') {
      lines.push(partHtml);
    }
  }
  // the log message element is with "white-space: break-spaces;", so use "\n" to break lines
  return lines.join('\n');
}

</script>

<style scoped>
.action-view-body {
  display: flex;
  height: calc(100vh - 266px); /* fine tune this value to make the main view has full height */
}

/* ================ */
/* action view header */

.action-view-header {
  margin: 20px 0px;
}

.action-info-summary {
  display: flex;
  align-items: center;
  margin-top: 1rem;
  justify-content: space-between;
}

.action-info-summary-title {
  display: flex;
}

.action-info-summary-title-text {
  font-size: 20px;
  margin: 0 0 0 5px;
  flex: 1;
}

.action-commit-summary {
  display: flex;
  gap: 5px;
  margin: 10px 0px 10px 25px;
}

/* ================ */
/* action view left */

.action-view-left {
  width: 30%;
  max-width: 400px;
  overflow-y: scroll;
}

.job-group-section .job-group-summary {
  margin: 5px 0;
  padding: 10px;
}

.job-artifacts-title {
  font-size: 18px;
  margin-top: 16px;
  padding: 16px 10px 0px 20px;
  border-top: 1px solid var(--color-secondary);
}

.job-artifacts-item {
  margin: 5px 0;
  padding: 6px;
}

.job-artifacts-list {
  padding-left: 12px;
  list-style: none;
}

.job-artifacts-icon {
  padding-right: 3px;
}

.job-brief-item {
  margin: 5px 0;
  padding: 10px;
  background: var(--color-info-bg);
  border-radius: 5px;
  text-decoration: none;
  display: flex;
  flex-wrap: nowrap;
  justify-content: space-between;
  align-items: center;
}

.job-brief-item:hover {
  background-color: var(--color-secondary);
}

.job-brief-item.selected {
  font-weight: var(--font-weight-bold);
  background-color: var(--color-secondary-dark-1);
}

.job-brief-item:first-of-type {
  margin-top: 0;
}

.job-brief-item .job-brief-rerun {
  cursor: pointer;
  transition: transform 0.2s;
}

.job-brief-item .job-brief-rerun:hover {
  transform: scale(130%);
}

.job-brief-item .job-brief-link {
  display: flex;
  width: 100%;
}

.job-brief-item .job-brief-link span {
  display: flex;
  align-items: center;
}

.job-brief-item .job-brief-link .job-brief-name {
  display: block;
  width: 70%;
  color: var(--color-text);
}

.job-brief-item .job-brief-link:hover {
  text-decoration: none;
}

.job-brief-item .job-brief-info {
  display: flex;
  align-items: center;
  width: 55px;
}

/* ================ */
/* action view right */

.action-view-right {
  flex: 1;
  background-color: var(--color-console-bg);
  color: var(--color-secondary-dark-2);
  max-height: 100%;
  width: 70%;
  display: flex;
  flex-direction: column;
}

.job-info-header {
  padding: 10px;
  border-bottom: 1px solid var(--color-grey);
}

.job-info-header .job-info-header-title {
  color: var(--color-console-fg);
  font-size: 16px;
  margin: 0;
}

.job-info-header .job-info-header-detail {
  color: var(--color-secondary-dark-3);
  font-size: 12px;
}

.job-step-container {
  max-height: 100%;
  overflow: auto;
}

.job-step-container .job-step-summary {
  cursor: pointer;
  padding: 5px 10px;
  display: flex;
  align-items: center;
  user-select: none;
}

.job-step-container .job-step-summary .step-summary-msg {
  flex: 1;
}

.job-step-container .job-step-summary .step-summary-duration {
  margin-left: 16px;
}

.job-step-container .job-step-summary:hover,
.job-step-container .job-step-summary.selected {
  color: var(--color-console-fg);
  background-color: var(--color-black-light);
  border-radius: 5px;
}

@media (max-width: 768px) {
  .action-view-body {
    flex-direction: column;
  }
  .action-view-left, .action-view-right {
    width: 100%;
  }

  .action-view-left {
    max-width: none;
    overflow-y: hidden;
  }
}
</style>

<style>
/* some elements are not managed by vue, so we need to use global style */
.job-status-rotate {
  animation: job-status-rotate-keyframes 1s linear infinite;
}

@keyframes job-status-rotate-keyframes {
  100% {
    transform: rotate(360deg);
  }
}

.job-step-section {
  margin: 10px;
}

.job-step-section .job-step-logs {
  font-family: monospace, monospace;
  margin: 8px 0px;
  font-size: 12px;
}

.job-step-section .job-step-logs .job-log-line {
  display: flex;
}

.job-step-section .job-step-logs .job-log-line:hover {
  color: var(--color-console-fg);
  background-color: var(--color-console-hover-bg);
}

.job-step-section .job-step-logs .job-log-line .line-num {
  width: 48px;
  color: var(--color-grey-light);
  text-align: right;
  user-select: none;
}

.job-step-section .job-step-logs .job-log-line .log-time {
  color: var(--color-grey-light);
  margin-left: 10px;
  white-space: nowrap;
}

.job-step-section .job-step-logs .job-log-line .log-msg {
  flex: 1;
  word-break: break-all;
  white-space: break-spaces;
  margin-left: 10px;
}

/* TODO: group support */

.job-log-group {

}
.job-log-group-summary {

}
.job-log-list {

}
</style>
