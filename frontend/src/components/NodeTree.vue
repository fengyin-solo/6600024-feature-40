<template>
  <div class="node-tree-container">
    <div class="tree-header">
      <h3 class="text-lg font-bold text-cyan-400">OPC-UA 节点树</h3>
      <el-tag :type="store.isConnected ? 'success' : 'danger'" size="small">
        {{ store.isConnected ? '已连接' : '未连接' }}
      </el-tag>
    </div>

    <el-tree
      :data="store.nodeTree"
      :props="treeProps"
      node-key="id"
      highlight-current
      default-expand-all
      @node-click="handleNodeClick"
      class="dark-tree"
    >
      <template #default="{ data }">
        <span class="custom-tree-node">
          <el-icon v-if="data.type === 'Object'" class="text-yellow-400">
            <Folder />
          </el-icon>
          <el-icon v-else-if="data.type === 'Variable'" class="text-green-400">
            <DataLine />
          </el-icon>
          <span class="node-label">{{ data.name }}</span>

          <span v-if="data.type === 'Variable'" class="status-badges">
            <el-tooltip
              v-if="data.quality"
              :content="'质量: ' + data.quality"
              placement="top"
              effect="dark"
            >
              <span
                class="status-badge"
                :class="[
                  'quality-badge',
                  data.quality === 'Good' ? 'quality-good' :
                  data.quality === 'Bad' ? 'quality-bad' : 'quality-uncertain'
                ]"
              >
                <el-icon :size="10">
                  <CircleCheck v-if="data.quality === 'Good'" />
                  <CircleClose v-else-if="data.quality === 'Bad'" />
                  <WarningFilled v-else />
                </el-icon>
              </span>
            </el-tooltip>

            <el-tooltip
              v-if="getAlarmSeverity(data.id)"
              :content="getAlarmTooltip(data.id)"
              placement="top"
              effect="dark"
            >
              <span
                class="status-badge alarm-badge"
                :class="getAlarmBadgeClass(getAlarmSeverity(data.id)!)"
              >
                <el-icon :size="10">
                  <BellFilled />
                </el-icon>
                <span
                  v-if="getNodeActiveAlarmCount(data.id) > 1"
                  class="alarm-count"
                >
                  {{ getNodeActiveAlarmCount(data.id) }}
                </span>
              </span>
            </el-tooltip>

            <el-tooltip
              :content="isNodeSubscribed(data.id) ? '已订阅' : '未订阅'"
              placement="top"
              effect="dark"
            >
              <span
                class="status-badge subscribe-badge"
                :class="isNodeSubscribed(data.id) ? 'subscribed' : 'unsubscribed'"
              >
                <el-icon :size="10">
                  <Connection v-if="isNodeSubscribed(data.id)" />
                  <Link v-else />
                </el-icon>
              </span>
            </el-tooltip>
          </span>

          <span v-if="data.type === 'Variable' && data.value !== undefined" class="node-value">
            {{ data.value }}{{ data.unit ? ' ' + data.unit : '' }}
          </span>
        </span>
      </template>
    </el-tree>

    <!-- 节点详情面板 -->
    <div v-if="store.selectedNode" class="node-detail-panel">
      <el-divider />
      <h4 class="text-sm font-bold text-cyan-300 mb-2">节点详情</h4>
      <el-descriptions :column="1" size="small" border class="dark-descriptions">
        <el-descriptions-item label="名称">{{ store.selectedNode.name }}</el-descriptions-item>
        <el-descriptions-item label="Node ID">{{ store.selectedNode.nodeId }}</el-descriptions-item>
        <el-descriptions-item label="类型">
          <el-tag size="small">{{ store.selectedNode.type }}</el-tag>
        </el-descriptions-item>
        <el-descriptions-item v-if="store.selectedNode.dataType" label="数据类型">
          {{ store.selectedNode.dataType }}
        </el-descriptions-item>
        <el-descriptions-item v-if="store.selectedNode.value !== undefined" label="当前值">
          <span class="text-green-400 font-mono">
            {{ store.selectedNode.value }}{{ store.selectedNode.unit ? ' ' + store.selectedNode.unit : '' }}
          </span>
        </el-descriptions-item>
        <el-descriptions-item v-if="store.selectedNode.quality" label="质量码">
          <el-tag
            :type="store.selectedNode.quality === 'Good' ? 'success' : 'danger'"
            size="small"
          >
            {{ store.selectedNode.quality }}
          </el-tag>
        </el-descriptions-item>
        <el-descriptions-item v-if="store.selectedNode.description" label="描述">
          {{ store.selectedNode.description }}
        </el-descriptions-item>
      </el-descriptions>

      <div class="mt-3 flex gap-2">
        <el-button
          v-if="store.selectedNode.type === 'Variable'"
          type="primary"
          size="small"
          @click="handleSubscribe"
        >
          {{ isSubscribed ? '取消订阅' : '订阅' }}
        </el-button>
        <el-button
          v-if="store.selectedNode.type === 'Variable'"
          type="info"
          size="small"
          @click="handleReadValue"
        >
          读取
        </el-button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import {
  Folder,
  DataLine,
  CircleCheck,
  CircleClose,
  WarningFilled,
  BellFilled,
  Connection,
  Link
} from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'
import { useOpcuaStore } from '../store/opcua'
import type { OPCUANode, AlarmEvent } from '../types'

const store = useOpcuaStore()

const treeProps = {
  children: 'children',
  label: 'name'
}

const isSubscribed = computed(() => {
  if (!store.selectedNode) return false
  return store.subscriptions.has(store.selectedNode.id)
})

function getAlarmSeverity(nodeId: string): AlarmEvent['severity'] | null {
  return store.getNodeHighestAlarmSeverity(nodeId)
}

function getNodeActiveAlarmCount(nodeId: string): number {
  return store.getNodeActiveAlarms(nodeId).length
}

function getAlarmTooltip(nodeId: string): string {
  const alarms = store.getNodeActiveAlarms(nodeId)
  if (alarms.length === 0) return ''
  if (alarms.length === 1) {
    return `[${alarms[0].severity}] ${alarms[0].message}`
  }
  return `${alarms.length} 条活动报警 · 最高: ${alarms[0].severity}`
}

function getAlarmBadgeClass(severity: AlarmEvent['severity']): string {
  const map: Record<AlarmEvent['severity'], string> = {
    Critical: 'alarm-critical',
    High: 'alarm-high',
    Medium: 'alarm-medium',
    Low: 'alarm-low',
    Info: 'alarm-info'
  }
  return map[severity] || 'alarm-info'
}

function isNodeSubscribed(nodeId: string): boolean {
  return store.isNodeSubscribed(nodeId)
}

function handleNodeClick(data: OPCUANode) {
  store.selectNode(data)
}

function handleSubscribe() {
  if (!store.selectedNode) return
  if (isSubscribed.value) {
    store.removeSubscription(store.selectedNode.id)
    ElMessage.success(`已取消订阅: ${store.selectedNode.name}`)
  } else {
    store.addSubscription(store.selectedNode.id)
    ElMessage.success(`已订阅: ${store.selectedNode.name}`)
  }
}

function handleReadValue() {
  if (!store.selectedNode) return
  ElMessage.success(`${store.selectedNode.name} = ${store.selectedNode.value} ${store.selectedNode.unit || ''}`)
}
</script>

<style scoped>
.node-tree-container {
  height: 100%;
  overflow-y: auto;
  padding: 12px;
}

.tree-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}

.custom-tree-node {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 13px;
  flex: 1;
  overflow: hidden;
}

.node-label {
  white-space: nowrap;
}

.status-badges {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  margin-left: 4px;
}

.status-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  border: 1.5px solid;
  background: rgba(0, 0, 0, 0.3);
  cursor: pointer;
  transition: all 0.2s ease;
  position: relative;
  flex-shrink: 0;
}

.status-badge:hover {
  transform: scale(1.15);
  filter: brightness(1.2);
}

.quality-badge {
  border-color: #67c23a;
  color: #67c23a;
}

.quality-badge.quality-good {
  border-color: #67c23a;
  color: #67c23a;
  background: rgba(103, 194, 58, 0.15);
  box-shadow: 0 0 4px rgba(103, 194, 58, 0.4);
}

.quality-badge.quality-bad {
  border-color: #f56c6c;
  color: #f56c6c;
  background: rgba(245, 108, 108, 0.15);
  box-shadow: 0 0 4px rgba(245, 108, 108, 0.4);
  animation: pulse-danger 2s ease-in-out infinite;
}

.quality-badge.quality-uncertain {
  border-color: #e6a23c;
  color: #e6a23c;
  background: rgba(230, 162, 60, 0.15);
  box-shadow: 0 0 4px rgba(230, 162, 60, 0.4);
}

.alarm-badge {
  border-color: #f56c6c;
  color: #fff;
  animation: alarm-blink 1.5s ease-in-out infinite;
}

.alarm-badge.alarm-critical {
  border-color: #ef4444;
  background: rgba(239, 68, 68, 0.9);
  color: #fff;
  box-shadow: 0 0 6px rgba(239, 68, 68, 0.6);
  animation: alarm-blink-critical 0.8s ease-in-out infinite;
}

.alarm-badge.alarm-high {
  border-color: #f97316;
  background: rgba(249, 115, 22, 0.85);
  color: #fff;
  box-shadow: 0 0 6px rgba(249, 115, 22, 0.5);
  animation: alarm-blink 1.2s ease-in-out infinite;
}

.alarm-badge.alarm-medium {
  border-color: #eab308;
  background: rgba(234, 179, 8, 0.85);
  color: #1f2937;
  box-shadow: 0 0 5px rgba(234, 179, 8, 0.5);
}

.alarm-badge.alarm-low {
  border-color: #3b82f6;
  background: rgba(59, 130, 246, 0.8);
  color: #fff;
}

.alarm-badge.alarm-info {
  border-color: #6b7280;
  background: rgba(107, 114, 128, 0.7);
  color: #fff;
  animation: none;
}

.alarm-count {
  position: absolute;
  top: -4px;
  right: -4px;
  font-size: 9px;
  font-weight: bold;
  background: #ef4444;
  color: #fff;
  min-width: 12px;
  height: 12px;
  line-height: 12px;
  padding: 0 3px;
  border-radius: 6px;
  text-align: center;
  border: 1px solid #fff;
}

.subscribe-badge {
  border-color: #6b7280;
  color: #6b7280;
  background: rgba(107, 114, 128, 0.1);
}

.subscribe-badge.subscribed {
  border-color: #06b6d4;
  color: #06b6d4;
  background: rgba(6, 182, 212, 0.15);
  box-shadow: 0 0 4px rgba(6, 182, 212, 0.4);
}

.subscribe-badge.unsubscribed {
  opacity: 0.5;
}

@keyframes alarm-blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.65; }
}

@keyframes alarm-blink-critical {
  0%, 100% {
    opacity: 1;
    box-shadow: 0 0 6px rgba(239, 68, 68, 0.6);
  }
  50% {
    opacity: 0.8;
    box-shadow: 0 0 14px rgba(239, 68, 68, 0.9);
  }
}

@keyframes pulse-danger {
  0%, 100% { box-shadow: 0 0 4px rgba(245, 108, 108, 0.4); }
  50% { box-shadow: 0 0 10px rgba(245, 108, 108, 0.7); }
}

.node-value {
  margin-left: auto;
  font-family: monospace;
  font-size: 12px;
  color: #67c23a;
  padding-left: 8px;
}

.node-detail-panel {
  padding: 8px 0;
}

:deep(.el-tree) {
  background: transparent !important;
  color: #e0e0e0 !important;
}

:deep(.el-tree-node__content:hover) {
  background: rgba(6, 182, 212, 0.1) !important;
}

:deep(.el-tree-node.is-current > .el-tree-node__content) {
  background: rgba(6, 182, 212, 0.2) !important;
}

:deep(.el-descriptions) {
  --el-descriptions-item-bordered-label-background: #1f2937;
}
</style>
