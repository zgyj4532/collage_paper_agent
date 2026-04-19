<template>
  <view class="paper-list-page page-enter" :class="{ 'page-fade-out': isPageFadeOut }">
    <!-- 顶部标题栏 -->
    <header class="top-header">
      <view class="header-left">
        <view class="header-brand">
          <view class="header-brand-icon">
            <text class="material-symbols-outlined">description</text>
          </view>
          <view class="header-brand-text">
            <text class="header-brand-title">我的论文</text>
            <text class="header-brand-subtitle">论文管理系统</text>
          </view>
        </view>
      </view>
      <view class="header-right">
        <view class="profile-btn-wrapper student-profile-btn"
          @mouseenter="onUserCardEnter"
          @mouseleave="onUserCardLeave">
          <view class="profile-avatar">
            <text>{{ userNameInitial }}</text>
          </view>
          <view class="profile-info">
            <text class="profile-name">{{ userInfo.full_name || userInfo.name }}</text>
            <text class="profile-role">{{ userInfo.username || '学号未设置' }}</text>
          </view>
          
          <!-- 用户信息卡片 -->
          <view class="user-info-card student-user-card" :class="{ show: showUserCard }" @click.stop
            @mouseenter="onUserCardEnter"
            @mouseleave="onUserCardLeave">
            <view class="user-card-header">
              <view class="user-card-avatar">
                <text>{{ userNameInitial }}</text>
              </view>
              <view class="user-card-info">
                <text class="user-name">{{ userInfo.full_name || userInfo.name }}</text>
                <text class="user-id">学号：{{ userInfo.owner_id || userInfo.username || '未设置' }}</text>
                <view class="user-role-badge">学生</view>
              </view>
            </view>
            <view class="user-card-menu">
              <view class="user-card-menu-item" @click="openChangePassword">
                <text class="material-symbols-outlined">lock</text>
                <text>修改密码</text>
              </view>
              <view class="user-card-menu-item" @click="openAboutModal">
                <text class="material-symbols-outlined">info</text>
                <text>关于系统</text>
              </view>
              <view class="user-card-menu-item logout" @click="logout">
                <text class="material-symbols-outlined">logout</text>
                <text>退出登录</text>
              </view>
            </view>
          </view>
        </view>
      </view>
    </header>

    <view class="main-layout-wrapper">
    <view class="main-layout">
      <!-- 左侧边栏 -->
      <view class="sidebar">
        <view class="sidebar-title">论文统计</view>
        <view class="sidebar-content">
          <view class="stats-panel">
            <view class="stat-item">
              <view class="stat-icon material-symbols-outlined">description</view>
              <view class="stat-content">
                <text class="stat-label">论文总数</text>
                <text class="stat-value">{{ paperList.length }}</text>
              </view>
            </view>
            <view
              class="stat-item clickable agent-stat-item"
              :class="{ disabled: !sidebarAgentPaper }"
              @click="handleSidebarAgentUse"
            >
              <view class="stat-icon material-symbols-outlined">smart_toy</view>
              <view class="stat-content">
                <text class="agent-stat-title">智能体使用</text>
                <text class="agent-stat-desc">{{ sidebarAgentPaper ? '当前论文已就绪，点击进入' : '暂无论文可用' }}</text>
              </view>
            </view>
            <view class="stat-item clickable agent-permission-entry" @click="openAgentPermissionApplyModal">
              <view class="stat-icon material-symbols-outlined">verified_user</view>
              <view class="stat-content">
                <text class="agent-stat-title">智能体权限</text>
                <text class="agent-stat-desc">向管理员提交调用权限申请</text>
              </view>
            </view>
            
          </view>
          <view class="sidebar-menu">
            <view class="menu-item" @click="goToWorkbench">
              <view class="menu-icon material-symbols-outlined">home</view>
              <text class="menu-text">返回工作台</text>
            </view>
            <view class="menu-item" @click="goToAttachmentList">
              <view class="menu-icon material-symbols-outlined">attach_file</view>
              <text class="menu-text">附件列表</text>
            </view>
          </view>
        </view>
      </view>

      <!-- 右侧内容区 -->
      <view class="content-area" id="scrollContainer">
        <view class="content-wrapper">
        <!-- 加载中提示 -->
        <view v-if="loading" class="loading-container">
          <view class="loading-spinner"></view>
          <text class="loading-text">正在加载论文数据...</text>
        </view>

        <!-- 论文列表 -->
        <transition-group v-else name="paper-list" tag="view" class="paper-list">
          <view v-for="(paper, index) in filteredPapers" :key="paper.id" :id="'paper-' + paper.id" class="paper-card" :class="{ 'highlighted': highlightedPaperId === paper.id }" @click="viewPaper(paper)">
            <!-- 论文头部 -->
            <view class="paper-header">
              <view class="paper-info">
                <text class="paper-title">{{ paper.title }}<text class="paper-preview-hint"> 点击预览 👁</text></text>
                <text class="paper-meta">指导教师：{{ paper.teacher }} | 版本：v{{ paper.version }} | 论文ID：{{ paper.id }}</text>
              </view>
              <view class="paper-header-actions" @click.prevent.stop>
                <view class="refresh-status-btn" @click.prevent.stop="fetchAllPaperStatus">
                  <text class="refresh-text">获取状态</text>
                </view>
              </view>
            </view>

            <!-- 论文内容 -->
            <view class="paper-content-wrapper">
              <view class="paper-content">
                <view class="paper-detail">
                  <text class="detail-label">最后更新：</text>
                  <text class="detail-value">{{ paper.updateTime }}</text>
                </view>

                <!-- 状态进度条（三步：与侧栏展示桶一致，依据后端 status 映射） -->
                <view class="progress-section">
                  <view class="progress-track">
                    <view class="progress-step" :class="{ active: displayStep(paper.status) >= 1, completed: displayStep(paper.status) > 1 }">
                      <view class="step-circle">1</view>
                      <text class="step-text">待审阅</text>
                    </view>
                    <view class="progress-line" :class="{ active: displayStep(paper.status) >= 2 }">
                      <view class="line-flow" v-if="displayStep(paper.status) >= 2"></view>
                    </view>
                    <view class="progress-step" :class="{ active: displayStep(paper.status) >= 2, completed: displayStep(paper.status) > 2 }">
                      <view class="step-circle">2</view>
                      <text class="step-text">待修改</text>
                    </view>
                    <view class="progress-line" :class="{ active: displayStep(paper.status) >= 3 }">
                      <view class="line-flow" v-if="displayStep(paper.status) >= 3"></view>
                    </view>
                    <view class="progress-step" :class="{ active: displayStep(paper.status) >= 3 }">
                      <view class="step-circle">3</view>
                      <text class="step-text">已定稿</text>
                    </view>
                  </view>
                </view>

                <!-- 状态记录区域 -->
                <view class="notice-section" v-if="paper.statusHistory && paper.statusHistory.length > 0">
                  <view class="notice-header">
                    <text class="notice-title">状态记录</text>
                  </view>
                  <view class="notice-list">
                    <view v-for="(record, nIdx) in paper.statusHistory.slice(0, 2)" :key="nIdx" class="notice-item">
                      <view class="notice-time">{{ record.time }}</view>
                      <view class="notice-content">{{ record.content }}</view>
                    </view>
                  </view>
                </view>
              </view>
            </view>

            <!-- 操作按钮 -->
            <view class="paper-actions">
              <button class="action-btn annotation-btn" @click.stop="viewAnnotations(paper)">
                <text class="btn-icon material-symbols-outlined">chat</text>
                <text>查看批注</text>
              </button>
              <button 
                class="action-btn review-btn" 
                :class="{ 'disabled': !canViewReview(paper.status) }"
                @click.stop="viewReview(paper)"
              >
                <text class="btn-icon material-symbols-outlined">fact_check</text>
                <text>查看审阅</text>
              </button>
              <button class="action-btn download-btn" @click.stop="downloadPaper(paper)">
                <text class="btn-icon material-symbols-outlined">download</text>
                <text>下载论文</text>
              </button>
              <button 
                class="action-btn update-btn" 
                :class="{ 'disabled': !canUpdatePaper(paper.status) }"
                @click.stop="handleUpdateClick(paper)"
              >
                <text class="btn-icon material-symbols-outlined">update</text>
                <text>更新论文</text>
              </button>
              <button class="action-btn delete-btn" @click.stop="deletePaper(paper.id, paper.title)">
                <text class="btn-icon material-symbols-outlined">delete</text>
                <text>删除</text>
              </button>
            </view>
          </view>
        </transition-group>

        <!-- 空状态 -->
        <view v-if="filteredPapers.length === 0 && !loading" class="empty-container">
          <text class="empty-icon material-symbols-outlined">inbox</text>
          <text class="empty-text">暂无论文数据</text>
          <text class="empty-subtext">请先上传论文以查看列表</text>
        </view>
      </view>
    </view>

    <!-- 批注详情模态框 -->
    <view class="annotation-modal" v-if="showAnnotationModal" @click="closeAnnotationModal">
      <view class="annotation-modal-content" @click.stop>
        <view class="annotation-modal-header">
          <view class="annotation-header-left"></view>
          <text class="annotation-modal-title">论文批注</text>
          <view class="annotation-header-right">
            <text class="annotation-modal-close material-symbols-outlined" @click="closeAnnotationModal">close</text>
          </view>
        </view>
        <view class="annotation-modal-body">
          <view class="annotation-paper-info">
            <text class="annotation-paper-title">{{ currentPaper?.title }}</text>
            <text class="annotation-paper-meta">指导教师：{{ currentPaper?.teacher }} | 版本：v{{ currentPaper?.version }}</text>
          </view>
          <view class="annotation-list" v-if="annotations.length > 0">
            <view 
              class="annotation-item" 
              v-for="(annotation, index) in annotations"
              :key="index"
              :class="{ 'processed': annotation.processed, 'unprocessed': !annotation.processed }"
            >
              <view class="annotation-header">
                <view class="annotation-source">
                  <text class="source-icon material-symbols-outlined">{{ annotation.source === 'AI' ? 'smart_toy' : 'person' }}</text>
                  <text class="source-text">{{ annotation.source === 'AI' ? 'AI智能批注' : '教师批注' }}</text>
                </view>
                <view class="annotation-status" :class="annotation.processed ? 'status-processed' : 'status-unprocessed'">
                  <text>{{ annotation.processed ? '已处理' : '未处理' }}</text>
                </view>
              </view>
                        
              <!-- 选中内容 -->
              <view class="annotation-section" v-if="annotation.content.selected">
                <view class="section-title">选中内容</view>
                <view class="section-content selected-content">{{ annotation.content.selected }}</view>
              </view>
                        
              <!-- 批注内容 -->
              <view class="annotation-section" v-if="annotation.content.annotation">
                <view class="section-title">批注</view>
                <view class="section-content annotation-content-text">{{ annotation.content.annotation }}</view>
              </view>
                        
              <!-- 建议 -->
              <view class="annotation-section" v-if="annotation.content.suggestion">
                <view class="section-title">建议</view>
                <view class="section-content suggestion-content">{{ annotation.content.suggestion }}</view>
              </view>
                        
              <view class="annotation-footer">
                <text class="annotation-time">{{ annotation.time }}</text>
              </view>
            </view>
          </view>
          <view class="annotation-empty" v-else>
            <text class="empty-icon material-symbols-outlined">chat</text>
            <text class="empty-text">暂无批注</text>
            <text class="empty-subtext">该论文暂时还没有批注信息</text>
          </view>
        </view>
      </view>
    </view>
    </view>
    </view>

    <!-- 更新论文弹窗 -->
    <view class="update-modal" v-if="showUpdateModal" @click.self="closeUpdateModal">
      <view class="update-modal-content">
        <view class="update-modal-header">
          <text class="update-modal-title">更新论文</text>
          <text class="update-modal-close" @click="closeUpdateModal">×</text>
        </view>
        <view class="update-modal-body">
          <view class="update-paper-info" v-if="updatePaperItem">
            <text class="update-paper-title">{{ updatePaperItem.title }}</text>
            <text class="update-paper-current-version">当前版本：v{{ updatePaperItem.version }}</text>
          </view>
          
          <view class="update-modal-tips">
            <text>支持格式：.docx（Word文档）</text>
            <text>文件大小：≤100MB</text>
          </view>
          
          <!-- 版本号输入 -->
          <view class="version-input-section">
            <text class="input-label">新版本号：</text>
            <input 
              class="version-input" 
              v-model="updateVersion" 
              type="text"
              placeholder="请输入新版本号（如 v2.0）"
            />
            <text class="version-hint">必须大于当前版本 v{{ updatePaperItem?.version }}</text>
          </view>
          
          <button class="update-modal-btn" @click="chooseUpdateFile">选择文件</button>
          <view v-if="updateSelectedFile" class="update-modal-file-info">
            <text class="update-modal-file-name">{{ updateSelectedFile.name }}</text>
            <text class="update-modal-file-size">{{ formatFileSize(updateSelectedFile.size) }}</text>
          </view>
          <button 
            class="update-modal-submit-btn" 
            :disabled="!updateSelectedFile || !updateVersion" 
            @click="submitUpdatePaper"
          >
            提交更新
          </button>
        </view>
      </view>
    </view>
    
    <!-- Word文档预览组件 -->
    <WordPreview
      :visible="showWordPreview"
      :fileUrl="previewFileUrl"
      :title="previewTitle"
      @close="closeWordPreview"
      @download="handlePreviewDownload"
      @error="handlePreviewError"
    />
    
    <!-- 确认弹窗组件 -->
    <ConfirmModal
      :visible="showConfirmModal"
      :title="confirmModalTitle"
      :content="confirmModalContent"
      @confirm="handleConfirmModalConfirm"
      @cancel="handleConfirmModalCancel"
    />

  <!-- 审阅详情弹窗 - 放在最外层避免受父元素transform影响 -->
  <view class="review-modal" v-if="showReviewModal" @click="closeReviewModal">
    <view class="review-modal-content" @click.stop>
      <view class="review-modal-header">
        <text class="review-modal-title">论文审阅详情</text>
        <text class="review-modal-close material-symbols-outlined" @click="closeReviewModal">close</text>
      </view>
      <view class="review-modal-body">
        <view class="review-paper-info" v-if="currentReview">
          <text class="review-paper-title">{{ currentReview.paperTitle }}</text>
          <text class="review-paper-meta">论文ID：{{ currentReview.paperId }}</text>
        </view>
        <view class="review-content-section" v-if="currentReview">
          <view class="review-section-title">审阅内容</view>
          <view class="review-content-text">{{ currentReview.reviewContent }}</view>
        </view>
        <view class="review-time-section" v-if="currentReview && currentReview.reviewTime">
          <text class="review-time-label">审阅时间：</text>
          <text class="review-time-value">{{ currentReview.reviewTime }}</text>
        </view>
        <view class="review-empty" v-else>
          <text class="empty-icon material-symbols-outlined">fact_check</text>
          <text class="empty-text">暂无审阅内容</text>
        </view>
      </view>
    </view>
  </view>
    
  <!-- 修改密码弹窗 -->
  <view v-if="showPasswordModal" class="modal-backdrop" @click.self="closePasswordModal">
    <view class="modal-content password-modal-content student-user-panel-modal">
      <view class="user-panel-header">
        <text class="user-panel-title">修改密码</text>
        <text class="user-panel-close" @click="closePasswordModal">×</text>
      </view>
      <view class="user-panel-body">
        <view class="user-panel-form-item">
          <text class="user-panel-form-label">当前密码</text>
          <input 
            class="user-panel-form-input"
            type="password" 
            v-model="passwordForm.currentPassword"
            placeholder="请输入当前密码"
          />
        </view>
        <view class="user-panel-form-item">
          <text class="user-panel-form-label">新密码</text>
          <input 
            class="user-panel-form-input"
            type="password" 
            v-model="passwordForm.newPassword"
            placeholder="请输入新密码（至少6位）"
          />
        </view>
        <view class="user-panel-form-item">
          <text class="user-panel-form-label">确认新密码</text>
          <input 
            class="user-panel-form-input"
            type="password" 
            v-model="passwordForm.confirmPassword"
            placeholder="请再次输入新密码"
          />
        </view>
        <view class="user-panel-form-tips" v-if="!passwordError">
          <text class="user-panel-tips-text">密码修改成功后需要重新登录</text>
        </view>
        <view class="user-panel-form-tips error-tips" v-else>
          <text class="user-panel-tips-text error-text">{{ passwordError }}</text>
        </view>
      </view>
      <view class="user-panel-footer">
        <view class="user-panel-btn user-panel-btn-cancel" @click="closePasswordModal">取消</view>
        <view class="user-panel-btn user-panel-btn-confirm" @click="submitChangePassword">确认修改</view>
      </view>
    </view>
  </view>
  
  <!-- 智能体权限提示弹窗 -->
  <view v-if="showAgentPermissionModal" class="modal-backdrop" @click.self="closeAgentPermissionModal">
    <view class="modal-content agent-permission-modal">
      <view class="modal-header agent-permission-header">
        <text class="modal-title">智能体使用权限</text>
        <text class="modal-close" @click="closeAgentPermissionModal">×</text>
      </view>
      <view class="modal-body agent-permission-modal-body">
        <view class="agent-permission-icon">
          <text class="material-symbols-outlined">lock_person</text>
        </view>
        <view class="agent-permission-title">当前未开通智能体权限</view>
        <view class="agent-permission-student">学号 {{ agentPermissionStudentId || '未获取' }}</view>
        <view class="agent-permission-desc">当前暂未开通智能体使用权限，无法使用该功能。</view>
        <view class="agent-permission-tip">
          <text class="material-symbols-outlined">campaign</text>
          <text>如需此权限，请通过“权限开通”向管理员申请。</text>
        </view>
      </view>
      <view class="modal-footer agent-permission-footer">
        <view class="btn btn-confirm" @click="closeAgentPermissionModal">我知道了</view>
      </view>
    </view>
  </view>

  <!-- 智能体使用提示弹窗 -->
  <view v-if="showAgentUseModal" class="modal-backdrop" @click.self="!agentUseModal.loading && closeAgentUseModal()">
    <view class="modal-content agent-use-modal">
      <view class="modal-header agent-use-header">
        <text class="modal-title">智能体使用</text>
        <text class="modal-close" @click="!agentUseModal.loading && closeAgentUseModal()">&#215;</text>
      </view>
      <view class="modal-body agent-use-modal-body">

        <!-- 确认提交阶段（ready） -->
        <template v-if="agentUseModal.ready">
          <view class="agent-use-icon">
            <text class="material-symbols-outlined">smart_toy</text>
          </view>
          <view class="agent-use-title">智能体审核</view>
          <view class="agent-use-student">学号 {{ agentUseModal.studentId || '未获取' }}</view>
          <view class="agent-use-paper">当前论文：{{ agentUseModal.paperTitle || '当前论文' }}</view>
          <view class="agent-use-tip">
            <text class="material-symbols-outlined">info</text>
            <text>确认后将对当前论文提交智能体审核，系统将在后台进行分析。</text>
          </view>
        </template>

        <!-- 加载中 -->
        <template v-else-if="agentUseModal.loading">
          <view class="agent-use-icon agent-use-icon-loading">
            <text class="material-symbols-outlined agent-spin">progress_activity</text>
          </view>
          <view class="agent-use-title">正在提交智能体审核...</view>
          <view class="agent-use-student">学号 {{ agentUseModal.studentId || '未获取' }}</view>
          <view class="agent-use-paper">当前论文：{{ agentUseModal.paperTitle || '当前论文' }}</view>
          <view class="agent-use-desc">正在调用智能体接口，请稍候...</view>
        </template>

        <!-- 调用失败 -->
        <template v-else-if="agentUseModal.error">
          <view class="agent-use-icon agent-use-icon-error">
            <text class="material-symbols-outlined">error_outline</text>
          </view>
          <view class="agent-use-title agent-use-title-error">调用失败</view>
          <view class="agent-use-student">学号 {{ agentUseModal.studentId || '未获取' }}</view>
          <view class="agent-use-paper">当前论文：{{ agentUseModal.paperTitle || '当前论文' }}</view>
          <view class="agent-use-error-msg">{{ agentUseModal.error }}</view>
        </template>

        <!-- 提交成功 -->
        <template v-else>
          <view class="agent-use-icon">
            <text class="material-symbols-outlined">smart_toy</text>
          </view>
          <view class="agent-use-title">智能体审核已提交</view>
          <view class="agent-use-student">学号 {{ agentUseModal.studentId || '未获取' }}</view>
          <view class="agent-use-paper">当前论文：{{ agentUseModal.paperTitle || '当前论文' }}</view>
          <view v-if="agentUseModal.version" class="agent-use-desc">论文版本：{{ agentUseModal.version }}</view>
          <view v-if="agentUseModal.taskId" class="agent-use-tip agent-use-success-tip">
            <text class="material-symbols-outlined">task_alt</text>
            <text v-if="agentUseModal.alreadySubmitted">该版本已提交过，任务ID：{{ agentUseModal.taskId }}</text>
            <text v-else>任务已创建，任务ID：{{ agentUseModal.taskId }}</text>
          </view>
          <!-- 后端 MySQL写入异常但实际成功的提示 -->
          <view v-if="agentUseModal.verifiedByQuery" class="agent-use-tip agent-use-verified-tip">
            <text class="material-symbols-outlined">info</text>
            <text>审核任务已创建，请前往「智能体报告」查看审核进度与结果。</text>
          </view>
          <view v-if="agentUseModal.alreadySubmitted" class="agent-use-tip agent-use-warn-tip">
            <text class="material-symbols-outlined">info</text>
            <text>同一版本论文只能提交一次，如需重新提交请更新论文版本。</text>
          </view>
          <view v-if="!agentUseModal.taskId" class="agent-use-tip">
            <text class="material-symbols-outlined">auto_awesome</text>
            <text>智能体已接收任务，正在后台处理中。</text>
          </view>
        </template>

      </view>
      <!-- 底部按钮：ready 阶段显示「取消 + 提交审查」，其他阶段显示「知道了」 -->
      <view class="modal-footer agent-use-footer">
        <template v-if="agentUseModal.ready">
          <view class="btn btn-cancel" @click="closeAgentUseModal">取消</view>
          <view class="btn btn-confirm agent-use-submit-btn" @click="doSubmitAgentAudit">
            <text class="material-symbols-outlined">send</text>
            <text>提交审查</text>
          </view>
        </template>
        <view
          v-else
          class="btn btn-confirm"
          :class="{ disabled: agentUseModal.loading }"
          @click="!agentUseModal.loading && closeAgentUseModal()"
        >{{ agentUseModal.loading ? '处理中...' : '知道了' }}</view>
      </view>
    </view>
  </view>

  <!-- 智能体权限申请弹窗 -->
  <view v-if="showAgentPermissionApplyModal" class="modal-backdrop" @click.self="closeAgentPermissionApplyModal">
    <view class="modal-content agent-request-modal">
      <view class="modal-header agent-request-header">
        <text class="modal-title">智能体权限</text>
        <text class="modal-close" @click="closeAgentPermissionApplyModal">×</text>
      </view>
      <view v-if="!agentPermissionApply.submitted" class="modal-body agent-request-modal-body">
        <view class="agent-request-icon">
          <text class="material-symbols-outlined">admin_panel_settings</text>
        </view>
        <view class="agent-request-title">申请智能体调用权限</view>
        <view class="agent-request-desc">请输入管理员 ID，提交后系统会向对应管理员发送权限申请消息。</view>
        <view class="agent-request-student">当前学号：{{ userInfo.username || '未获取' }}</view>
        <view class="agent-request-form">
          <text class="agent-request-label">管理员ID</text>
          <input
            class="agent-request-input"
            v-model.trim="agentPermissionApply.adminId"
            type="text"
            maxlength="30"
            placeholder="请输入管理员ID，例如 a123"
            placeholder-class="agent-request-placeholder"
          />
          <text v-if="agentPermissionApply.error" class="agent-request-error">{{ agentPermissionApply.error }}</text>
        </view>
        <view class="agent-request-tip">
          <text class="material-symbols-outlined">info</text>
          <text>请先确认管理员 ID 填写正确，再提交权限申请。</text>
        </view>
      </view>
      <view v-else class="modal-body agent-request-modal-body success-state">
        <view class="agent-request-icon success">
          <text class="material-symbols-outlined">mark_email_read</text>
        </view>
        <view class="agent-request-title">申请已提交</view>
        <view class="agent-request-desc">{{ agentPermissionApply.message || '权限申请已提交，等待管理员审批。' }}</view>
        <view class="agent-request-badge">管理员ID：{{ agentPermissionApply.submittedAdminId || '--' }}</view>
        <view v-if="agentPermissionApply.studentName" class="agent-request-badge">姓名：{{ agentPermissionApply.studentName }}</view>
        <view class="agent-request-badge">学号：{{ agentPermissionApply.studentNumber || userInfo.username || '--' }}</view>
        <view v-if="agentPermissionApply.messageId" class="agent-request-tip success-tip">
          <text class="material-symbols-outlined">alternate_email</text>
          <text>消息编号：{{ agentPermissionApply.messageId }}</text>
        </view>
      </view>
      <view class="modal-footer agent-request-footer">
        <template v-if="!agentPermissionApply.submitted">
          <view class="btn btn-cancel" @click="closeAgentPermissionApplyModal">取消</view>
          <view class="btn btn-confirm" :class="{ disabled: agentPermissionApply.submitting }" @click="submitAgentPermissionApply">
            {{ agentPermissionApply.submitting ? '提交中...' : '提交申请' }}
          </view>
        </template>
        <view v-else class="btn btn-confirm" @click="closeAgentPermissionApplyModal">我知道了</view>
      </view>
    </view>
  </view>
  </view>

  <!-- 智能体审核报告弹窗 -->
  <view v-if="showAgentReportModal" class="modal-backdrop agent-report-backdrop" @click.self="closeAgentReportModal">
    <view class="modal-content agent-report-modal">
      <!-- 弹窗头部 -->
      <view class="modal-header agent-report-header">
        <view class="agent-report-header-left">
          <view class="agent-report-header-icon">
            <text class="material-symbols-outlined">summarize</text>
          </view>
          <view class="agent-report-header-text">
            <text class="agent-report-header-title">智能体审核报告</text>
            <text v-if="agentReport.version" class="agent-report-header-sub">版本 {{ agentReport.version }}</text>
          </view>
        </view>
        <text class="modal-close" @click="closeAgentReportModal">&#215;</text>
      </view>

      <!-- 弹窗主体 -->
      <view class="modal-body agent-report-body">

        <!-- 全局加载中 -->
        <view v-if="agentReport.loading" class="agent-report-loading">
          <text class="material-symbols-outlined agent-spin">progress_activity</text>
          <text class="agent-report-loading-text">正在查询任务信息...</text>
        </view>

        <!-- 无任务ID错误 -->
        <view v-else-if="agentReport.taskError && !agentReport.taskId" class="agent-report-empty">
          <view class="agent-report-empty-icon">
            <text class="material-symbols-outlined">search_off</text>
          </view>
          <text class="agent-report-empty-title">未找到审核任务</text>
          <text class="agent-report-empty-desc">{{ agentReport.taskError }}</text>
          <text class="agent-report-empty-tip">请先在「智能体使用」中提交论文审核。</text>
        </view>

        <!-- 有任务ID：展示任务状态 + 报告 -->
        <template v-else>
          <!-- 任务元信息卡片 -->
          <view class="agent-report-task-card">
            <view class="agent-report-task-row">
              <text class="agent-report-task-label">任务 ID</text>
              <text class="agent-report-task-value agent-report-mono">{{ agentReport.taskId || '—' }}</text>
            </view>
            <view class="agent-report-task-row">
              <text class="agent-report-task-label">状态</text>
              <view class="agent-report-status-badge" :class="'agent-report-status-' + (agentReport.taskStatus || 'unknown').toLowerCase()">
                <text class="material-symbols-outlined agent-spin" v-if="agentReport.taskStatus === 'processing' || agentReport.taskStatus === 'running' || agentReport.taskStatus === 'pending'">progress_activity</text>
                <text class="material-symbols-outlined" v-else-if="isAgentTaskDone(agentReport.taskStatus)">check_circle</text>
                <text class="material-symbols-outlined" v-else-if="isAgentTaskFailed(agentReport.taskStatus)">error_outline</text>
                <text class="material-symbols-outlined" v-else>schedule</text>
                <text>{{ agentReportTaskStatusLabel(agentReport.taskStatus) }}</text>
              </view>
            </view>
            <view v-if="agentReport.taskProgress != null" class="agent-report-task-row">
              <text class="agent-report-task-label">进度</text>
              <view class="agent-report-progress-wrap">
                <view class="agent-report-progress-bar">
                  <view class="agent-report-progress-fill" :style="{ width: agentReport.taskProgress + '%' }"></view>
                </view>
                <text class="agent-report-progress-pct">{{ agentReport.taskProgress }}%</text>
              </view>
            </view>
          </view>

          <!-- 任务失败提示 -->
          <view v-if="agentReport.taskError" class="agent-report-error-tip">
            <text class="material-symbols-outlined">warning</text>
            <text>{{ agentReport.taskError }}</text>
          </view>

          <!-- 报告内容区 -->
          <view v-if="agentReport.payload" class="agent-report-content">

            <!-- 概览行 -->
            <view class="agent-report-summary-row">
              <view v-if="agentReportIssuesCount != null" class="agent-report-stat-chip">
                <text class="material-symbols-outlined">bug_report</text>
                <text>共 {{ agentReportIssuesCount }} 个问题</text>
              </view>
              <view v-if="agentReportReferenceVerification.length" class="agent-report-stat-chip chip-teal">
                <text class="material-symbols-outlined">import_contacts</text>
                <text>参考文献 {{ agentReportReferenceVerification.length }} 条</text>
              </view>
              <view v-if="agentReportChunkReviews.length" class="agent-report-stat-chip chip-purple">
                <text class="material-symbols-outlined">segment</text>
                <text>{{ agentReportChunkReviews.length }} 个分段</text>
              </view>
            </view>

            <!-- 参考文献核查 -->
            <view v-if="agentReportReferenceVerification.length" class="agent-report-section">
              <view class="agent-report-section-title">
                <text class="material-symbols-outlined">import_contacts</text>
                <text>参考文献核查</text>
                <text class="agent-report-section-count">{{ agentReportReferenceVerification.length }} 条</text>
              </view>
              <view class="agent-report-ref-list">
                <view
                  v-for="(ref, rIdx) in agentReportReferenceVerification"
                  :key="'ref-' + rIdx"
                  class="agent-report-ref-item"
                  :class="{ 'ref-issue': ref.has_issue || ref.hasIssue }"
                >
                  <view class="agent-report-ref-top">
                    <text class="agent-report-ref-num">[{{ ref.index ?? ref.ref_index ?? (rIdx + 1) }}]</text>
                    <text class="agent-report-ref-text">{{ ref.raw || ref.text || ref.content || '' }}</text>
                  </view>
                  <text v-if="ref.remark || ref.note" class="agent-report-ref-remark">{{ ref.remark || ref.note }}</text>
                </view>
              </view>
            </view>

            <!-- 分段问题 -->
            <view v-if="agentReportChunkReviews.length" class="agent-report-section">
              <view class="agent-report-section-title">
                <text class="material-symbols-outlined">segment</text>
                <text>分段问题</text>
                <text class="agent-report-section-count">{{ agentReportChunkReviewsFiltered.length }} 段</text>
              </view>

              <!-- 问题类型筛选标签 -->
              <view v-if="agentReportIssueTypes.length > 1" class="agent-report-filter-bar">
                <text
                  class="agent-report-filter-tag"
                  :class="{ active: agentReport.issueTypeFilter === 'all' }"
                  @click="agentReport.issueTypeFilter = 'all'"
                >全部</text>
                <text
                  v-for="tp in agentReportIssueTypes"
                  :key="tp"
                  class="agent-report-filter-tag"
                  :class="{ active: agentReport.issueTypeFilter === tp }"
                  @click="agentReport.issueTypeFilter = tp"
                >{{ formatAgentIssueType(tp) }}</text>
              </view>

              <!-- 分段列表 -->
              <view
                v-for="(chunk, cIdx) in agentReportChunkReviewsFiltered"
                :key="'chunk-' + cIdx"
                class="agent-report-chunk-item"
              >
                <view class="agent-report-chunk-head">
                  <text class="material-symbols-outlined">article</text>
                  <text>{{ agentReportChunkHeadLabel(chunk, cIdx) }}</text>
                  <text class="agent-report-chunk-count">{{ agentReportChunkIssueCount(chunk) }} 条</text>
                </view>
                <view
                  v-for="(iss, iIdx) in agentReportChunkIssuesFiltered(chunk)"
                  :key="'iss-' + cIdx + '-' + iIdx"
                  class="agent-report-issue"
                >
                  <view class="agent-report-issue-type-badge">{{ formatAgentIssueType(iss.issue_type || iss.issueType) }}</view>
                  <text class="agent-report-issue-msg">{{ iss.message || iss.msg || '' }}</text>
                  <text v-if="iss.suggestion" class="agent-report-issue-sug">建议：{{ iss.suggestion }}</text>
                </view>
                <view v-if="agentReportChunkIssuesFiltered(chunk).length === 0" class="agent-report-chunk-empty">
                  <text>该分段无符合筛选条件的问题</text>
                </view>
              </view>
            </view>

            <!-- 报告为空时的占位 -->
            <view v-if="!agentReportChunkReviews.length && !agentReportReferenceVerification.length" class="agent-report-no-issues">
              <text class="material-symbols-outlined">check_circle</text>
              <text>未检测到明显问题，论文质量良好！</text>
            </view>
          </view>

          <!-- 任务未完成时的提示 -->
          <view v-else-if="!agentReport.taskStatus || (!isAgentTaskDone(agentReport.taskStatus) && !isAgentTaskFailed(agentReport.taskStatus))" class="agent-report-pending-hint">
            <text class="material-symbols-outlined">hourglass_top</text>
            <text>智能体正在分析中，请点击下方「查询进度」刷新状态。</text>
          </view>

        </template>
      </view>

      <!-- 弹窗底部 -->
      <view class="modal-footer agent-report-footer">
        <view class="btn btn-cancel" @click="closeAgentReportModal">关闭</view>
        <view
          v-if="agentReport.taskId && !agentReport.payload"
          class="btn btn-confirm agent-report-query-btn"
          :class="{ disabled: agentReport.queryLoading }"
          @click="!agentReport.queryLoading && fetchAgentReportTaskStatus()"
        >
          <text v-if="agentReport.queryLoading" class="material-symbols-outlined agent-spin">progress_activity</text>
          <text v-else class="material-symbols-outlined">refresh</text>
          <text>{{ agentReport.queryLoading ? '查询中...' : '查询进度' }}</text>
        </view>
        <view
          v-if="agentReport.payload"
          class="btn btn-confirm agent-report-refresh-btn"
          :class="{ disabled: agentReport.queryLoading }"
          @click="!agentReport.queryLoading && fetchAgentReportTaskStatus()"
        >
          <text class="material-symbols-outlined">refresh</text>
          <text>刷新报告</text>
        </view>
      </view>
    </view>
  </view>

</template>

<script>
import { getPaperList, deletePaper, getAnnotationsByPaperId, updatePaperVersion, getPaperDetail, updatePaperStatus, getTeacherByStudentId, getPaperReview, checkAgentPermission, requestAgentPermission, changePassword, callAgentAudit, getAgentTaskByPaper, getAgentTask, getAgentReport } from '../../api/student.js';
import { getUserId, isValidUserId, checkLogin } from '../../utils/auth.js';
import { AsyncQueue, debounce } from '../../utils/performance.js';
import {
	mapBackendStatusToStudentDisplayBucket,
	studentDisplayStatusLabel,
	studentDisplayStep,
	STUDENT_DISPLAY_BUCKETS
} from '../../utils/studentPaperDisplayStatus.js';

import WordPreview from '../../components/WordPreview.vue';
import ConfirmModal from '../../components/ConfirmModal.vue';
import { config } from '../../api/config.js';
import { clearLoginState } from '../../utils/auth.js';

export default {
  components: {
    WordPreview,
    ConfirmModal
  },
  data() {
    return {
      // 页面淡出状态（跳转动画）
      isPageFadeOut: false,
      /** 论文展示三态（与 utils/studentPaperDisplayStatus 一致，供模板侧栏统计） */
      displayStatusBuckets: STUDENT_DISPLAY_BUCKETS,
      paperList: [],
      loading: true,
      // Word预览相关
      showWordPreview: false,
      previewFileUrl: '',
      previewTitle: '',
      previewPaper: null,
      currentFilter: 'all',
      // 高亮的论文ID（从工作台跳转过来）
      highlightedPaperId: null,
      // 用户信息
      userInfo: {
        name: '学生',
        full_name: '',
        username: '',
        college: '学院'
      },
      // 论文列表刷新key
      paperListKey: 0,
      showAnnotationModal: false,
      currentPaper: null,
      annotations: [],
      // 更新论文相关
      showUpdateModal: false,
      updatePaperItem: null,
      updateVersion: '',
      updateSelectedFile: null,
      // 确认弹窗相关
      showConfirmModal: false,
      confirmModalTitle: '',
      confirmModalContent: '',
      confirmModalCallback: null,
      // 审阅弹窗相关
      showReviewModal: false,
      currentReview: null,
      backupPaperList: [
        {
          id: 1,
          title: "基于深度学习的图像识别研究",
          version: 2.0,
          teacher: "张教授",
          status: "reviewed",
          statusText: "待修改",
          updateTime: "2026-01-18 15:30",
          fileUrl: "https://example.com/papers/deep-learning-image-recognition.pdf",
          notices: [
            { time: '2026-01-18 15:30', content: '论文状态：已审阅' },
            { time: '2026-01-17 10:20', content: '论文状态：待审阅' }
          ]
        },
        {
          id: 2,
          title: "大数据在医疗领域的应用研究",
          version: 1.0,
          teacher: "李教授",
          status: "pending_review",
          statusText: "待审阅",
          updateTime: "2026-01-10 09:15",
          fileUrl: "https://example.com/papers/big-data-healthcare.pdf",
          notices: [
            { time: '2026-01-10 09:15', content: '论文状态：待审阅' }
          ]
        },
        {
          id: 3,
          title: "计算机视觉技术及应用",
          version: 1.5,
          teacher: "王教授",
          status: "finalized",
          statusText: "已定稿",
          updateTime: "2026-01-20 16:40",
          fileUrl: "https://example.com/papers/computer-vision-applications.pdf",
          notices: [
            { time: '2026-01-20 16:40', content: '论文状态：已定稿' }
          ]
        }
      ],
      // 用户卡片
      showUserCard: false,
      // 修改密码弹窗
      showPasswordModal: false,
      passwordForm: {
        currentPassword: '',
        newPassword: '',
        confirmPassword: ''
      },
      passwordError: '',
      // 智能体权限提示弹窗
      showAgentPermissionModal: false,
      agentPermissionStudentId: '',
      showAgentUseModal: false,
      agentUseModal: {
        studentId: '',
        paperTitle: '当前论文',
        ready: false,
        paper: null,
        loading: false,
        taskId: '',
        version: '',
        error: '',
        alreadySubmitted: false,
        verifiedByQuery: false
      },
      showAgentPermissionApplyModal: false,
      agentPermissionApply: {
        adminId: '',
        error: '',
        submitting: false,
        submitted: false,
        message: '',
        submittedAdminId: '',
        studentNumber: '',
        studentName: '',
        messageId: ''
      },
      // 智能体报告弹窗
      showAgentReportModal: false,
      agentReport: {
        loading: false,
        taskId: null,
        taskStatus: '',
        taskProgress: null,
        taskError: '',
        payload: null,
        queryLoading: false,
        issueTypeFilter: 'all',
        paper: null,
        version: ''
      }
    };
  },
  computed: {
    filteredPapers() {
      // 始终返回新数组以确保Vue响应式更新
      if (this.currentFilter === 'all') {
        return [...this.paperList];
      }
      const f = this.currentFilter;
      const buckets = Object.values(STUDENT_DISPLAY_BUCKETS);
      if (buckets.includes(f)) {
        return this.paperList.filter(
          paper => mapBackendStatusToStudentDisplayBucket(paper.status) === f
        );
      }
      return this.paperList.filter(paper => paper.status === f);
    },
    sidebarAgentPaper() {
      return this.paperList[0] || null;
    },
    // 用户名首字母
    userNameInitial() {
      const displayName = this.userInfo.full_name || this.userInfo.name || '学';
      return displayName.charAt(0);
    },
    // 智能体报告 - 问题总数
    agentReportIssuesCount() {
      const p = this.agentReport.payload;
      if (!p) return null;
      if (p.issues_count != null) return p.issues_count;
      if (p.issuesCount != null) return p.issuesCount;
      const ar = p.ai_review || p.aiReview;
      const sum = ar && ar.summary;
      if (sum && typeof sum === 'object' && sum.chunk_issue_count != null) return sum.chunk_issue_count;
      return null;
    },
    // 智能体报告 - 分段审查列表
    agentReportChunkReviews() {
      const p = this.agentReport.payload;
      if (!p) return [];
      const list = p.chunk_reviews || p.chunkReviews;
      if (Array.isArray(list) && list.length > 0) return list;
      const parsed = (p.parse_result && p.parse_result.data) || p.parse_result || {};
      const sections = parsed.sections;
      return Array.isArray(sections) ? sections : [];
    },
    // 智能体报告 - 问题类型列表
    agentReportIssueTypes() {
      const types = new Set();
      for (const chunk of this.agentReportChunkReviews) {
        const issues = chunk.issues || chunk.llm_issues || chunk.local_issues || [];
        for (const iss of issues) {
          const t = iss.issue_type || iss.issueType;
          if (t) types.add(t.toLowerCase());
        }
      }
      return Array.from(types);
    },
    // 智能体报告 - 筛选后的分段列表
    agentReportChunkReviewsFiltered() {
      const all = this.agentReportChunkReviews;
      const filter = this.agentReport.issueTypeFilter;
      if (!filter || filter === 'all') return all;
      return all.filter(chunk => {
        const issues = chunk.issues || chunk.llm_issues || chunk.local_issues || [];
        return issues.some(iss => (iss.issue_type || iss.issueType || '').toLowerCase() === filter);
      });
    },
    // 智能体报告 - 参考文献核查
    agentReportReferenceVerification() {
      const p = this.agentReport.payload;
      if (!p) return [];
      const r = p.reference_verification || p.referenceVerification;
      return Array.isArray(r) ? r : [];
    }
  },
  async onLoad() {
    if (!checkLogin({ redirect: true, message: '请先登录' })) {
      this.loading = false;
      return;
    }
    // 加载用户信息
    this.loadUserInfo();
    // 首次加载显示 loading
    await this.getPaperList(true);
    // 检查是否有目标论文ID（从工作台跳转过来）
    this.checkAndNavigateToTargetPaper();
    this._skipNextShow = true;
  },
  onShow() {
    // 重置淡出状态
    this.isPageFadeOut = false;
    
    if (this._skipNextShow) {
      this._skipNextShow = false;
      return;
    }
    // 从后台返回时不显示 loading，使用缓存快速显示
    this.getPaperList(false);
  },
  
  onUnload() {
    // 清理高亮定时器
    if (this._highlightTimer) {
      clearTimeout(this._highlightTimer);
      this._highlightTimer = null;
    }
    if (this._scrollTimer) {
      clearTimeout(this._scrollTimer);
      this._scrollTimer = null;
    }
    if (this._userCardShowTimer) {
      clearTimeout(this._userCardShowTimer);
      this._userCardShowTimer = null;
    }
    if (this._userCardHideTimer) {
      clearTimeout(this._userCardHideTimer);
      this._userCardHideTimer = null;
    }
  },
  
  methods: {
    // 用户卡片鼠标进入（延迟显示）
    onUserCardEnter() {
      if (this._userCardHideTimer) {
        clearTimeout(this._userCardHideTimer);
        this._userCardHideTimer = null;
      }
      this._userCardShowTimer = setTimeout(() => {
        this.showUserCard = true;
      }, 300);
    },
    
    // 用户卡片鼠标离开（延迟隐藏）
    onUserCardLeave() {
      if (this._userCardShowTimer) {
        clearTimeout(this._userCardShowTimer);
        this._userCardShowTimer = null;
      }
      this._userCardHideTimer = setTimeout(() => {
        this.showUserCard = false;
      }, 1000);
    },
    
    // 打开修改密码弹窗
    openChangePassword() {
      this.showUserCard = false;
      this.passwordForm = { currentPassword: '', newPassword: '', confirmPassword: '' };
      this.passwordError = '';
      this.showPasswordModal = true;
    },

    openAboutModal() {
      this.showUserCard = false;
      uni.showToast({
        title: '当前为论文管理系统v1.0版本',
        icon: 'none'
      });
    },

    openAgentPermissionModal(studentId) {
      this.agentPermissionStudentId = studentId || '';
      this.showAgentPermissionModal = true;
    },

    closeAgentPermissionModal() {
      this.showAgentPermissionModal = false;
      this.agentPermissionStudentId = '';
    },

    openAgentUseModal(studentId, paperTitle) {
      this.agentUseModal = {
        studentId: studentId || '',
        paperTitle: paperTitle || '当前论文'
      };
      this.showAgentUseModal = true;
    },

    closeAgentUseModal() {
      this.showAgentUseModal = false;
      this.agentUseModal = {
        studentId: '',
        paperTitle: '当前论文',
        ready: false,
        paper: null,
        loading: false,
        taskId: '',
        version: '',
        error: '',
        alreadySubmitted: false,
        verifiedByQuery: false
      };
    },

    resetAgentPermissionApplyState() {
      this.agentPermissionApply = {
        adminId: '',
        error: '',
        submitting: false,
        submitted: false,
        message: '',
        submittedAdminId: '',
        studentNumber: '',
        studentName: '',
        messageId: ''
      };
    },

    openAgentPermissionApplyModal() {
      this.showUserCard = false;
      this.resetAgentPermissionApplyState();
      this.showAgentPermissionApplyModal = true;
    },

    // 打开智能体报告弹窗
    async openAgentReportModal() {
      const paper = this.sidebarAgentPaper;
      if (!paper) {
        uni.showToast({ title: '暂无论文可查询', icon: 'none' });
        return;
      }

      // 获取版本
      let version = '';
      const detailCache = uni.getStorageSync(`paperDetail_${paper.id}`);
      if (detailCache && detailCache.version) {
        version = detailCache.version;
      } else {
        const rawVersion = paper.version || '1.0';
        version = rawVersion.startsWith('v') ? rawVersion : `v${rawVersion}`;
      }

      // 从缓存读取 task_id
      let taskId = null;
      const taskInfoCache = uni.getStorageSync(`agentTaskInfo_${paper.id}_${version}`);
      const auditCache = uni.getStorageSync(`agentAudit_${paper.id}_${version}`);
      if (taskInfoCache && taskInfoCache.task_id) {
        taskId = taskInfoCache.task_id;
      } else if (auditCache && auditCache.taskId) {
        taskId = auditCache.taskId;
      }

      // 初始化弹窗状态
      this.agentReport = {
        loading: !taskId,
        taskId,
        taskStatus: '',
        taskProgress: null,
        taskError: '',
        payload: null,
        queryLoading: false,
        issueTypeFilter: 'all',
        paper,
        version
      };
      this.showAgentReportModal = true;

      // 没有 task_id 时，调用 by-paper 接口获取
      if (!taskId) {
        try {
          const res = await getAgentTaskByPaper(paper.id, version);
          const data = this.unwrapAgentPayload(res);
          taskId = data?.task_id ?? data?.id ?? null;
          if (taskId) {
            this.agentReport.taskId = taskId;
            try { uni.setStorageSync(`agentTaskInfo_${paper.id}_${version}`, { task_id: taskId, paper_id: paper.id, version }); } catch (e) {}
          } else {
            this.agentReport.loading = false;
            this.agentReport.taskError = '未找到该版本的审核任务，请先在「智能体使用」中提交审核。';
            return;
          }
        } catch (e) {
          this.agentReport.loading = false;
          this.agentReport.taskError = e?.message || '查询任务失败';
          return;
        }
      }

      // 有 task_id 后查询任务状态
      this.agentReport.loading = false;
      await this.fetchAgentReportTaskStatus();
    },

    closeAgentReportModal() {
      this.showAgentReportModal = false;
      this.agentReport = {
        loading: false, taskId: null, taskStatus: '', taskProgress: null,
        taskError: '', payload: null, queryLoading: false, issueTypeFilter: 'all',
        paper: null, version: ''
      };
    },

    // 查询任务进度（底部按钟触发）
    async fetchAgentReportTaskStatus() {
      if (!this.agentReport.taskId) return;
      this.agentReport.queryLoading = true;
      this.agentReport.taskError = '';
      try {
        const raw = await getAgentTask(this.agentReport.taskId);
        const t = this.unwrapAgentPayload(raw);
        this.agentReport.taskStatus = t?.status || '';
        this.agentReport.taskProgress = t?.progress != null ? Number(t.progress) : null;
        const errMsg = t?.error_message ?? t?.errorMessage;
        if (errMsg) this.agentReport.taskError = String(errMsg);
        if (this.isAgentTaskDone(this.agentReport.taskStatus)) {
          await this.fetchAgentReportData();
        } else if (this.isAgentTaskFailed(this.agentReport.taskStatus)) {
          this.agentReport.taskError = this.agentReport.taskError || '智能体分析失败';
        }
      } catch (e) {
        this.agentReport.taskError = e?.message || '查询进度失败';
      } finally {
        this.agentReport.queryLoading = false;
      }
    },

    // 拉取报告 JSON 数据
    async fetchAgentReportData() {
      if (!this.agentReport.taskId) return;
      this.agentReport.queryLoading = true;
      try {
        const raw = await getAgentReport(this.agentReport.taskId);
        const rep = this.unwrapAgentPayload(raw);
        this.agentReport.payload = rep && typeof rep === 'object' ? rep : null;
        if (!this.agentReport.payload) throw new Error('报告数据为空');
      } catch (e) {
        this.agentReport.taskError = e?.message || '获取报告失败';
      } finally {
        this.agentReport.queryLoading = false;
      }
    },

    // 解包 API 响应数据
    unwrapAgentPayload(res) {
      if (res == null || typeof res !== 'object') return res;
      const d = res.data;
      if (d && typeof d === 'object' && !Array.isArray(d) && (
        d.task_id != null || d.status != null || d.paper_id != null ||
        d.chunk_reviews != null || d.parse_result != null || d.ai_review != null || d.issues_count != null
      )) return d;
      if (d && typeof d === 'object' && d.data && typeof d.data === 'object') {
        const inner = d.data;
        if (inner.chunk_reviews != null || inner.parse_result != null ||
            Array.isArray(inner.sections) || inner.ai_review != null || inner.issues_count != null)
          return inner;
      }
      return res;
    },

    isAgentTaskDone(status) {
      const s = (status || '').toLowerCase();
      return s === 'done' || s === 'completed' || s === 'success';
    },
    isAgentTaskFailed(status) {
      const s = (status || '').toLowerCase();
      return s === 'failed' || s === 'error';
    },
    agentReportTaskStatusLabel(status) {
      const map = {
        pending: '排队中', processing: '分析中', running: '分析中',
        done: '已完成', completed: '已完成', success: '已完成',
        failed: '失败', error: '失败'
      };
      return map[(status || '').toLowerCase()] || status || '—';
    },
    agentReportChunkIssueCount(chunk) {
      if (!chunk) return 0;
      if (chunk.issue_count != null) return Number(chunk.issue_count);
      const arr = chunk.issues || chunk.llm_issues || chunk.local_issues || [];
      return Array.isArray(arr) ? arr.length : 0;
    },
    agentReportChunkIssuesFiltered(chunk) {
      if (!chunk) return [];
      const a = chunk.issues || chunk.llm_issues;
      const raw = (Array.isArray(a) && a.length > 0) ? a : (Array.isArray(chunk.local_issues) ? chunk.local_issues : []);
      const filter = this.agentReport.issueTypeFilter;
      if (!filter || filter === 'all') return raw;
      return raw.filter(iss => (iss.issue_type || iss.issueType || '').toLowerCase() === filter);
    },
    agentReportChunkHeadLabel(chunk, cIdx) {
      const sid = chunk?.section_id ?? chunk?.sectionId ?? cIdx + 1;
      return `段 ${sid}`;
    },
    formatAgentIssueType(type) {
      const map = {
        'format': '格式问题', 'typo': '拼写/错别字', 'logic': '逻辑问题',
        'reference': '参考文献', 'structure': '结构问题', 'language': '语言表达',
        'content': '内容问题', 'data': '数据问题', 'figure': '图表问题',
        'table': '表格问题', 'math': '公式问题', 'citation': '引用格式',
        'plagiarism': '重复内容', 'completeness': '内容完整性', 'other': '其他问题',
        'missing_required': '缺失必填项', 'placeholder_unchecked': '占位符未替换',
        'format_error': '格式错误', 'value_invalid': '字段値无效'
      };
      return map[(type || '').toLowerCase()] || type || '问题';
    },

    closeAgentPermissionApplyModal() {
      this.showAgentPermissionApplyModal = false;
      this.resetAgentPermissionApplyState();
    },

    async submitAgentPermissionApply() {
      if (this.agentPermissionApply.submitting) {
        return;
      }

      const adminId = String(this.agentPermissionApply.adminId || '').trim();
      if (!adminId) {
        this.agentPermissionApply.error = '请输入管理员ID';
        return;
      }

      this.agentPermissionApply.error = '';
      this.agentPermissionApply.submitting = true;
      uni.showLoading({ title: '提交中...', mask: true });

      try {
        const res = await requestAgentPermission(adminId);
        const payload = res?.data && typeof res.data === 'object' ? res.data : res;

        this.agentPermissionApply = {
          adminId,
          error: '',
          submitting: false,
          submitted: true,
          message: payload?.message || '权限申请已提交，等待管理员审批。',
          submittedAdminId: adminId,
          studentNumber: payload?.student_number || this.userInfo.username || '',
          studentName: payload?.student_name || this.userInfo.full_name || this.userInfo.name || '',
          messageId: payload?.message_id || ''
        };
      } catch (error) {
        this.agentPermissionApply.error =
          error?.message ||
          error?.detail ||
          error?.response?.message ||
          '提交申请失败，请稍后重试';
        this.agentPermissionApply.submitting = false;
      } finally {
        uni.hideLoading();
      }
    },
    
    // 关闭修改密码弹窗
    closePasswordModal() {
      this.showPasswordModal = false;
      this.passwordForm = { currentPassword: '', newPassword: '', confirmPassword: '' };
      this.passwordError = '';
    },
    
    // 提交修改密码
    async submitChangePassword() {
      const { currentPassword, newPassword, confirmPassword } = this.passwordForm;
      
      // 密码表单验证（生产环境不输出密码信息）
      if (!currentPassword) {
        this.passwordError = '请输入当前密码';
        return;
      }
      if (!newPassword) {
        this.passwordError = '请输入新密码';
        return;
      }
      if (newPassword.length < 6) {
        this.passwordError = '新密码长度不能少于6位';
        return;
      }
      if (newPassword !== confirmPassword) {
        this.passwordError = '两次输入的新密码不一致';
        return;
      }
      if (currentPassword === newPassword) {
        this.passwordError = '新密码不能与当前密码相同';
        return;
      }
      
      this.passwordError = '';
      
      try {
        if (process.env.NODE_ENV === 'development') {
          console.log('[ChangePassword] 开始修改密码');
        }
        
        uni.showLoading({ title: '修改中...', mask: true });
        
        const res = await changePassword({
          old_password: currentPassword,
          new_password: newPassword
        });
        
        uni.hideLoading();
        
        // 判断修改成功（后端返回 message 或 HTTP 状态码为 200）
        if (res && (res.message?.includes('成功') || res.code === 200)) {
          uni.showToast({ title: '密码修改成功，请重新登录', icon: 'success', duration: 2000 });
          this.closePasswordModal();
          setTimeout(() => {
            clearLoginState();
            uni.reLaunch({ url: '/pages/index/index' });
          }, 2000);
        } else {
          // 显示后端返回的错误信息
          this.passwordError = res?.detail || res?.message || '密码修改失败';
        }
      } catch (err) {
        uni.hideLoading();
        this.passwordError = err?.message || err?.detail || '密码修改失败，请检查输入后重试';
      }
    },
    
    // 退出登录
    logout() {
      this.showUserCard = false;
      clearLoginState();
      uni.removeStorageSync('rememberedUsername');
      uni.showToast({ title: '已退出登录', icon: 'success' });
      setTimeout(() => {
        uni.reLaunch({ url: '/pages/index/index' });
      }, 1000);
    },
    
    // 加载用户信息
    loadUserInfo() {
      try {
        let userInfo = uni.getStorageSync('userInfo');
        
        // 如果获取的是字符串，需要解析
        if (typeof userInfo === 'string') {
          userInfo = JSON.parse(userInfo);
        }
        
        if (userInfo) {
          // 优先使用 full_name，如果不存在则使用 name
          const displayName = userInfo.full_name || userInfo.name || '学生用户';
          this.userInfo.name = displayName;
          this.userInfo.full_name = userInfo.full_name || userInfo.name || '学生';
          this.userInfo.username = userInfo.username || '';
          this.userInfo.college = userInfo.college || '学院';
        }
      } catch (e) {
        console.error('获取用户信息失败:', e);
      }
    },
    
    // 检查并跳转到目标论文（从工作台跳转）
    async checkAndNavigateToTargetPaper() {
      try {
        const targetPaperInfo = uni.getStorageSync('targetPaperInfo');
        if (!targetPaperInfo || !targetPaperInfo.id) {
          return;
        }
        
        console.log('[论文列表] 检测到目标论文:', targetPaperInfo);
        
        // 清除存储的目标信息
        uni.removeStorageSync('targetPaperInfo');
        
        const targetPaperId = targetPaperInfo.id;
        const targetStatus = targetPaperInfo.status;
        
        // 等待论文列表渲染完成
        await this.$nextTick();
        
        // 检查目标论文是否在当前筛选条件下可见
        let targetPaper = this.filteredPapers.find(p => p.id === targetPaperId);
        
        // 如果不在当前筛选条件下，切换筛选条件
        if (!targetPaper && targetStatus) {
          const bucket = mapBackendStatusToStudentDisplayBucket(targetStatus);
          console.log('[论文列表] 论文不在当前筛选条件下，切换展示筛选:', targetStatus, '->', bucket);
          this.currentFilter = bucket;
          await this.$nextTick();
          targetPaper = this.filteredPapers.find(p => p.id === targetPaperId);
        }
        
        if (!targetPaper) {
          // 尝试在所有论文中查找
          targetPaper = this.paperList.find(p => p.id === targetPaperId);
          if (targetPaper) {
            // 切换到全部显示
            console.log('[论文列表] 切换到全部显示以查找论文');
            this.currentFilter = 'all';
            await this.$nextTick();
          }
        }
        
        if (targetPaper) {
          console.log('[论文列表] 找到目标论文:', targetPaper.title);

          // 设置高亮状态
          this.highlightedPaperId = targetPaperId;
          
          // 延迟滚动到目标论文位置（等待页面淡入动画完成）
          this._scrollTimer = setTimeout(() => {
            this._scrollTimer = null;
            this.scrollToPaper(targetPaperId);
          }, 450);
          
          // 3秒后取消高亮
          this._highlightTimer = setTimeout(() => {
            this._highlightTimer = null;
            this.highlightedPaperId = null;
          }, 3000);
        } else {
          uni.showToast({ title: '未找到对应论文', icon: 'none' });
        }
      } catch (e) {
        if (process.env.NODE_ENV === 'development') {
          console.error('[PaperList] 检查目标论文失败');
        }
      }
    },
    
    // 滚动到指定论文位置
    scrollToPaper(paperId) {
      // 使用 uni.createSelectorQuery 获取元素位置
      const query = uni.createSelectorQuery().in(this);
      
      // 同时获取容器和目标元素的位置
      query.select('#scrollContainer').boundingClientRect();
      query.select('#scrollContainer').scrollOffset();
      query.select(`#paper-${paperId}`).boundingClientRect();
      
      query.exec((res) => {
        console.log('[论文列表] 滚动查询结果:', res);
        
        if (res && res[0] && res[2]) {
          const containerRect = res[0];  // 容器位置
          const scrollOffset = res[1];    // 容器当前滚动位置
          const targetRect = res[2];      // 目标元素位置
          
          if (!targetRect) {
            console.warn('[论文列表] 未找到目标元素:', paperId);
            return;
          }
          
          // 获取容器高度（可视区域高度）
          const containerHeight = containerRect.height;
          
          // 计算目标元素相对于容器的位置
          // targetRect.top 是元素相对于视口顶部的位置
          // containerRect.top 是容器相对于视口顶部的位置
          // 所以元素相对于容器顶部的位置 = targetRect.top - containerRect.top + 当前滚动位置
          const relativeTop = targetRect.top - containerRect.top + scrollOffset.scrollTop;
          
          // 计算目标滚动位置：使元素在容器中垂直居中
          const cardHeight = targetRect.height;
          const targetScrollTop = relativeTop - (containerHeight / 2) + (cardHeight / 2);
          
          // 确保滚动位置不小于0
          const finalScrollTop = Math.max(0, targetScrollTop);
          
          console.log('[论文列表] 滚动计算:', {
            containerHeight,
            relativeTop,
            cardHeight,
            currentScrollTop: scrollOffset.scrollTop,
            targetScrollTop: finalScrollTop
          });
          
          // 获取容器 DOM 元素并设置滚动位置
          // #ifdef H5
          const container = document.getElementById('scrollContainer');
          if (container) {
            container.scrollTo({
              top: finalScrollTop,
              behavior: 'smooth'
            });
            console.log('[论文列表] 已滚动到论文:', paperId);
          } else {
            console.warn('[论文列表] 未找到滚动容器');
          }
          // #endif
          
          // #ifndef H5
          // 非H5环境，尝试使用 uni.pageScrollTo
          uni.pageScrollTo({
            scrollTop: finalScrollTop,
            duration: 600
          });
          // #endif
        } else {
          console.warn('[论文列表] 无法获取元素位置，结果:', res);
        }
      });
    },
    
    // ========== 更新论文相关方法 ==========
    
    /**
     * 处理更新按钮点击
     * 如果状态不允许更新，显示对应状态的原因提示
     */
    handleUpdateClick(paper) {
      if (!this.canUpdatePaper(paper.status)) {
        const reason = this.getUpdateDisabledReason(paper.status);
        uni.showToast({ 
          title: reason, 
          icon: 'none',
          duration: 2500
        });
        return;
      }
      this.openUpdateModal(paper);
    },
    
    openUpdateModal(paper) {
      this.updatePaperItem = paper;
      this.updateVersion = '';
      this.updateSelectedFile = null;
      this.showUpdateModal = true;
    },
    
    closeUpdateModal() {
      this.showUpdateModal = false;
      this.updatePaperItem = null;
      this.updateVersion = '';
      this.updateSelectedFile = null;
    },
    
    chooseUpdateFile() {
      // H5 平台使用 input 元素选择文件
      // #ifdef H5
      const input = document.createElement('input');
      input.type = 'file';
      input.accept = '.docx,application/vnd.openxmlformats-officedocument.wordprocessingml.document';
      input.onchange = (e) => {
        const file = e.target.files[0];
        if (file) {
          this.updateSelectedFile = {
            name: file.name,
            size: file.size,
            path: file  // H5 下传递交 File 对象
          };
        }
      };
      input.click();
      // #endif
      
      // 小程序/移动端使用 uni.chooseMessageFile
      // #ifndef H5
      uni.chooseMessageFile({
        count: 1,
        type: 'file',
        extension: ['.docx'],
        success: (res) => {
          const file = res.tempFiles[0];
          this.updateSelectedFile = {
            name: file.name,
            size: file.size,
            path: file.path
          };
        },
        fail: (err) => {
          console.error('选择文件失败:', err);
          uni.showToast({ title: '选择文件失败', icon: 'none' });
        }
      });
      // #endif
    },
    
    formatFileSize(size) {
      if (size < 1024) return size + ' B';
      if (size < 1024 * 1024) return (size / 1024).toFixed(2) + ' KB';
      return (size / (1024 * 1024)).toFixed(2) + ' MB';
    },
    
    async submitUpdatePaper() {
      if (!this.updateSelectedFile || !this.updateVersion) {
        uni.showToast({ title: '请选择文件并输入版本号', icon: 'none' });
        return;
      }
      
      // 验证版本号格式
      const versionRegex = /^v\d+\.\d+$/;
      if (!versionRegex.test(this.updateVersion)) {
        uni.showToast({ title: '版本号格式不正确（如 v2.0）', icon: 'none' });
        return;
      }
      
      // 验证新版本是否大于当前版本
      const currentVersion = parseFloat(this.updatePaperItem.version);
      const newVersion = parseFloat(this.updateVersion.replace('v', ''));
      if (newVersion <= currentVersion) {
        uni.showToast({ title: `新版本必须大于当前版本 v${this.updatePaperItem.version}`, icon: 'none' });
        return;
      }
      
      // 关闭弹窗前保存所有需要的值（关闭后 updatePaperItem 会被清空）
      const paperId = this.updatePaperItem.id;
      const filePath = this.updateSelectedFile.path;
      const version = this.updateVersion;
      // 保存当前状态，用于判断是否需要触发状态回退
      const currentStatus = (this.updatePaperItem.status || '').trim();
      
      // 先关闭弹窗，再显示 loading
      this.closeUpdateModal();
      uni.showLoading({ title: '更新中...', mask: true });
      
      try {
        const res = await updatePaperVersion(paperId, filePath, version);
        console.log('[提交更新] 版本更新成功:', res);
        
        // 状态流转：仅当当前状态为"待更新"时，才将状态回退为"已更新"
        // 后端权限规则：待更新 → 学生可设为已更新
        const NEED_STATUS_UPDATE = ['待更新', 'pending_update', 'update_pending'];
        if (NEED_STATUS_UPDATE.includes(currentStatus) || NEED_STATUS_UPDATE.includes(currentStatus.toLowerCase())) {
          try {
            await updatePaperStatus(paperId, { status: '已更新' });
            console.log('[提交更新] 状态已更新为"已更新"，原状态:', currentStatus);
          } catch (statusErr) {
            console.error('[提交更新] 状态更新失败，原状态:', currentStatus, '错误:', statusErr);
            // 状态更新失败不阻断主流程，刷新列表后显示最新状态
          }
        } else {
          console.log('[提交更新] 当前状态无需触发状态变更:', currentStatus);
        }
        
        uni.hideLoading();
        uni.showToast({ title: '更新成功', icon: 'success' });
        
        // 刷新列表以显示最新状态
        this.getPaperList();
      } catch (err) {
        uni.hideLoading();
        console.error('[提交更新] 失败:', err);
        uni.showToast({ title: err.message || '更新失败', icon: 'none' });
      }
    },
    
    // 刷新所有论文状态（并行执行，带防抖）
    async fetchAllPaperStatus(forceRefresh = false) {
      // 防抖检查：5秒内不重复刷新
      const now = Date.now();
      if (!forceRefresh && this._lastStatusRefresh && (now - this._lastStatusRefresh < 5000)) {
        uni.showToast({ title: '刷新太频繁，请稍后再试', icon: 'none' });
        return;
      }
      this._lastStatusRefresh = now;
      
      if (this.paperList.length === 0) {
        uni.showToast({ title: '暂无论文', icon: 'none' });
        return;
      }
      
      this.loading = true;
      
      try {
        // 使用队列控制并发数，避免同时发起过多请求
        const queue = new AsyncQueue(3); // 最多3个并发
        const statusPromises = this.paperList.map(paper => 
          queue.add(() => 
            getPaperDetail(paper.id)
              .then(res => ({ id: paper.id, status: res?.status }))
              .catch(() => ({ id: paper.id, status: null }))
          )
        );
        
        const results = await Promise.all(statusPromises);
        const statusMap = {};
        let updatedCount = 0;
        
        results.forEach(({ id, status }) => {
          if (status) {
            // 标准化状态值
            const normalizedStatus = (status || '').toLowerCase().trim();
            const index = this.paperList.findIndex(p => p.id === id);
            if (index !== -1) {
              // 只在状态变化时更新
              if (this.paperList[index].status !== normalizedStatus) {
                this.$set(this.paperList[index], 'status', normalizedStatus);
                this.$set(this.paperList[index], 'statusText', this.formatStatusText(normalizedStatus));
                updatedCount++;
              }
            }
            statusMap[id] = normalizedStatus;
          }
        });
        
        this.savePaperStatusToStorage(statusMap);
        if (updatedCount > 0) {
          this.paperListKey++;
        }
        uni.showToast({ 
          title: updatedCount > 0 ? `已更新${updatedCount}篇论文状态` : '状态已是最新', 
          icon: 'success' 
        });
      } catch (err) {
        console.error('刷新状态失败:', err);
        uni.showToast({ title: '刷新状态失败', icon: 'none' });
      } finally {
        this.loading = false;
      }
    },
    
    goToWorkbench() {
      this.isPageFadeOut = true;
      setTimeout(() => {
        uni.navigateTo({ url: '/pages/student/workbench' });
      }, 300);
    },
    goToAttachmentList() {
      this.isPageFadeOut = true;
      setTimeout(() => {
        uni.navigateTo({ url: '/pages/student/attachmentList' });
      }, 300);
    },
    
    // ========== 论文状态本地存储相关方法 ==========
    /**
     * 保存论文状态到本地存储
     */
    savePaperStatusToStorage(statusMap) {
      const userId = getUserId();
      if (!userId) return;
      
      const storageKey = `paper_status_${userId}`;
      const data = {
        timestamp: Date.now(),
        statusMap: statusMap
      };
      uni.setStorageSync(storageKey, data);
    },
    
    /**
     * 从本地存储加载论文状态
     */
    loadPaperStatusFromStorage() {
      const userId = getUserId();
      if (!userId) return null;
      
      const storageKey = `paper_status_${userId}`;
      const data = uni.getStorageSync(storageKey);
      
      if (!data || !data.statusMap) return null;
      
      // 检查数据是否过期（7天）
      const now = Date.now();
      const expireTime = 7 * 24 * 60 * 60 * 1000;
      if (now - data.timestamp > expireTime) {
        uni.removeStorageSync(storageKey);
        return null;
      }
      
      return data.statusMap;
    },
    
    /**
     * 应用本地存储的状态到论文列表
     */
    applyStoredStatusToPaperList() {
      const statusMap = this.loadPaperStatusFromStorage();
      if (!statusMap) return;
      
      let hasUpdate = false;
      this.paperList.forEach((paper, index) => {
        const stored = statusMap[paper.id] != null ? statusMap[paper.id] : statusMap[String(paper.id)];
        if (!stored) return;

        const serverBucket = mapBackendStatusToStudentDisplayBucket(paper.status);
        // 列表接口已给出「待修改」或「已定稿」时，不要用本地旧缓存覆盖（否则会出现进度条与按钮不一致）
        if (
          serverBucket === STUDENT_DISPLAY_BUCKETS.PENDING_REVISION ||
          serverBucket === STUDENT_DISPLAY_BUCKETS.FINALIZED
        ) {
          return;
        }

        if (paper.status !== stored) {
          this.$set(this.paperList[index], 'status', stored);
          this.$set(this.paperList[index], 'statusText', this.formatStatusText(stored));
          hasUpdate = true;
        } else if (!paper.statusText) {
          this.$set(this.paperList[index], 'statusText', this.formatStatusText(stored));
          hasUpdate = true;
        }
      });
      
      if (hasUpdate) {
        this.paperListKey++;
      }
    },
    
    /**
     * 查询学生的指导教师工号，并更新论文列表中的教师显示
     * 接口返回 { teachers: [{id, teacher_id}] }，teacher_id 即教师工号
     * @param {string} studentUsername - 学生学号
     */
    async fetchAndApplyTeacherUsername(studentUsername) {
      try {
        const res = await getTeacherByStudentId(studentUsername);
        const teachers = res?.teachers || [];
        if (teachers.length === 0) return;
        
        // 构建 教师自增ID → 教师工号 的映射表
        const teacherMap = {};
        teachers.forEach(t => {
          teacherMap[t.id] = t.teacher_id; // t.teacher_id 是工号如 "t123"
        });
        
        // 更新论文列表中匹配的教师显示
        let hasUpdate = false;
        this.paperList.forEach((paper, index) => {
          // 取原始 teacher_id（数字），在映射表中查找工号
          const rawTeacherId = paper._rawTeacherId;
          if (rawTeacherId && teacherMap[rawTeacherId]) {
            this.$set(this.paperList[index], 'teacher', teacherMap[rawTeacherId]);
            hasUpdate = true;
          }
        });
        
        if (hasUpdate) this.paperListKey++;
      } catch (err) {
        console.error('[fetchAndApplyTeacherUsername] 查询教师工号失败:', err);
      }
    },
    
    // 获取缺少状态的论文详细状态（并行执行）
    async fetchAllPaperDetails(papers) {
      if (!papers || papers.length === 0) return;
          
      // 并行获取所有论文详情（包含状态和更新时间）
      const detailPromises = papers.map(paper =>
        getPaperDetail(paper.id)
          .then(res => ({ id: paper.id, detail: res }))
          .catch(() => ({ id: paper.id, detail: null }))
      );
          
      const results = await Promise.all(detailPromises);
          
      let hasUpdate = false;
      results.forEach(({ id, detail }) => {
        if (detail) {
          const index = this.paperList.findIndex(p => p.id === id);
          if (index !== -1) {
            // 标准化状态值（小写）
            const rawStatus = detail.status || '';
            const normalizedStatus = rawStatus.toLowerCase().trim();
            const oldStatus = this.paperList[index].status;
                
            // 调试日志
            console.log('[fetchAllPaperDetails] 论文ID:', id, '原始状态:', rawStatus, '-> 标准化:', normalizedStatus, '旧状态:', oldStatus);
                
            // 更新状态（如果后端返回的状态与当前不同）
            if (normalizedStatus && normalizedStatus !== oldStatus) {
              this.$set(this.paperList[index], 'status', normalizedStatus);
              this.$set(this.paperList[index], 'statusText', this.formatStatusText(normalizedStatus));
              hasUpdate = true;
            }
                
            // 更新更新时间
            if (detail.updated_at) {
              this.$set(this.paperList[index], 'updateTime', detail.updated_at);
            }
                
            // 缓存论文详情（含 version），供智能体调用时使用
            try { uni.setStorageSync(`paperDetail_${id}`, detail); } catch (e) {}

            // 构建并更新状态历史记录（基于后端返回的真实状态）
            const statusHistory = this.buildStatusHistory(detail);
            this.$set(this.paperList[index], 'statusHistory', statusHistory);
          }
        }
      });
          
      if (hasUpdate) {
        this.paperListKey++;
        this.savePaperStatusToStorageFromList();
      }
    },
    
    // 保留此方法用于兼容，实际逻辑已合并到 fetchAllPaperDetails
    async fetchMissingStatuses(papers) {
      return this.fetchAllPaperDetails(papers);
    },
    
    /**
     * 根据论文详情构建状态历史记录
     * @param {Object} detail - 论文详情对象
     * @returns {Array} 状态历史记录数组
     */
    buildStatusHistory(detail) {
      const history = [];
      const rawStatus = detail.status || '';
      const updateTime = detail.updated_at || detail.updateTime || new Date().toLocaleString('zh-CN');
      const displayStatus = studentDisplayStatusLabel(rawStatus);
      history.push({
        time: updateTime,
        content: `当前状态：${displayStatus}`
      });
      return history;
    },
    
    // 格式化状态文本（学生端统一三态展示，后端枚举不变）
    formatStatusText(status) {
      if (status == null || String(status).trim() === '') return '待审阅';
      return studentDisplayStatusLabel(status);
    },
    
    // 从列表保存状态到本地存储
    savePaperStatusToStorageFromList() {
      try {
        const statusMap = {};
        this.paperList.forEach(paper => {
          if (paper.status) {
            statusMap[paper.id] = paper.status;
          }
        });
        this.savePaperStatusToStorage(statusMap);
      } catch (error) {
        console.error('保存论文状态到本地存储失败:', error);
      }
    },
    
    getStatusCount(status) {
      // 标准化状态值进行比较（支持中英文）
      const normalizedTarget = this.normalizeStatus(status);
      return this.paperList.filter(paper => {
        const normalizedPaperStatus = this.normalizeStatus(paper.status);
        return normalizedPaperStatus === normalizedTarget;
      }).length;
    },

    /** 侧栏：按展示桶（ui_pending_review 等）统计数量 */
    getDisplayStatusCount(bucket) {
      return this.paperList.filter(
        p => mapBackendStatusToStudentDisplayBucket(p.status) === bucket
      ).length;
    },

    /** 模板用：三步进度当前步 1–3 */
    displayStep(status) {
      return studentDisplayStep(status);
    },
    
    /**
     * 标准化状态值（支持中英文互转）
     * 将各种状态格式统一为英文标准格式
     */
    normalizeStatus(status) {
      if (!status) return '';
      const normalized = String(status).toLowerCase().trim();
      
      // 中文状态映射到英文
      const statusMap = {
        '待审阅': 'pending_review',
        '已审阅': 'reviewed',
        '已更新': 'updated',
        '待更新': 'pending_update',
        '已定稿': 'finalized',
        '已上传': 'uploaded',
        '未提交': 'unsubmitted'
      };
      
      // 如果是中文，返回对应的英文
      if (statusMap[normalized]) {
        return statusMap[normalized];
      }
      
      // 如果是英文，标准化常见变体
      const englishMap = {
        'pending_review': 'pending_review',
        'review_pending': 'pending_review',
        'reviewing': 'pending_review',
        'under_review': 'pending_review',
        'in_review': 'pending_review',
        'reviewed': 'reviewed',
        'review_complete': 'reviewed',
        'feedback': 'reviewed',
        'updated': 'updated',
        'update_complete': 'updated',
        'pending_update': 'pending_update',
        'update_pending': 'pending_update',
        'finalized': 'finalized',
        'completed': 'finalized',
        'done': 'finalized',
        'uploaded': 'uploaded',
        'submitted': 'uploaded',
        'unsubmitted': 'unsubmitted',
        'draft': 'unsubmitted'
      };
      
      return englishMap[normalized] || normalized;
    },
    
    handleSidebarAgentUse() {
      if (!this.sidebarAgentPaper) {
        uni.showToast({
          title: '暂无论文可使用智能体',
          icon: 'none'
        });
        return;
      }

      this.handleAgentUse(this.sidebarAgentPaper);
    },
    
    /**
     * 判断学生是否可以更新论文
     * 与展示「待修改」同一规则：凡归入 ui_pending_revision 桶的后端状态均可更新（避免白名单漏写导致按钮灰显）
     */
    canUpdatePaper(status) {
      return (
        mapBackendStatusToStudentDisplayBucket(status) ===
        STUDENT_DISPLAY_BUCKETS.PENDING_REVISION
      );
    },
    
    /**
     * 获取论文不可更新时的原因提示
     * 根据不同状态返回对应的提示信息
     */
    getUpdateDisabledReason(status) {
      const bucket = mapBackendStatusToStudentDisplayBucket(status);
      const ns = this.normalizeStatus(status);
      console.log('[更新提示] 原始状态:', status, '-> 展示桶:', bucket, '-> normalize:', ns);
      if (bucket === STUDENT_DISPLAY_BUCKETS.FINALIZED) {
        return '论文已定稿，无法再进行更新';
      }
      if (ns === 'updated' || ns === 'update_complete') {
        return '论文已提交更新，请等待教师审阅';
      }
      if (
        ns === 'pending_review' ||
        ns === 'reviewing' ||
        ns === 'under_review' ||
        ns === 'in_review' ||
        ns === 'review_pending'
      ) {
        return '论文正在等待教师审阅，请耐心等待';
      }
      return '论文刚上传，请等待教师开始审阅';
    },
    
    // 判断是否可查看审阅（已审阅及之后的状态）
    canViewReview(status) {
      const normalizedStatus = (status || '').toLowerCase().trim();
      // 已审阅及之后的状态：已审阅、已更新、待更新、已定稿
      const canViewStatuses = [
        '已审阅', 'reviewed', 'review_complete',
        '已更新', 'updated', 'update_complete',
        '待更新', 'pending_update', 'update_pending',
        '已定稿', 'finalized', 'final'
      ];
      return canViewStatuses.includes(normalizedStatus);
    },
    
    // 查看审阅内容
    async viewReview(paper) {
      if (!this.canViewReview(paper.status)) {
        uni.showToast({
          title: '当前论文尚未审阅，暂无审阅内容',
          icon: 'none',
          duration: 2000
        });
        return;
      }
      
      uni.showLoading({ title: '加载审阅内容...' });
      
      try {
        const res = await getPaperReview(paper.id);
        uni.hideLoading();
        
        if (res && res.message && res.message.includes('成功')) {
          // 显示审阅内容弹窗
          this.currentReview = {
            paperId: paper.id,
            paperTitle: paper.title,
            reviewContent: res.review_content || '暂无审阅内容',
            reviewTime: res.review_time || res.created_at || '',
            updatedTime: res.updated_time || res.updated_at || '',
            teacherId: res.teacher_id || ''
          };
          this.showReviewModal = true;
        } else {
          uni.showToast({
            title: res.message || '获取审阅内容失败',
            icon: 'none'
          });
        }
      } catch (err) {
        uni.hideLoading();
        uni.showToast({
          title: err.message || '获取审阅内容失败',
          icon: 'none'
        });
      }
    },
    
    // 关闭审阅弹窗
    closeReviewModal() {
      this.showReviewModal = false;
      this.currentReview = null;
    },

    async handleAgentUse(paper) {
      if (!paper) {
        uni.showToast({
          title: '暂无论文可使用智能体',
          icon: 'none'
        });
        return;
      }

      const studentId = this.userInfo.username || uni.getStorageSync('userInfo')?.username || '';
      if (!studentId) {
        uni.showToast({
          title: '未获取到学生学号',
          icon: 'none'
        });
        return;
      }

      uni.showLoading({ title: '检查权限中...', mask: true });

      try {
        const res = await checkAgentPermission(studentId);
        uni.hideLoading();

        const payload = res?.data && typeof res.data === 'object' ? res.data : res;
        const permissionValue = Number(payload?.agent_permission);

        if (permissionValue === 1) {
          this.startAgentUseFlow(paper, studentId);
          return;
        }

        this.openAgentPermissionModal(studentId);
      } catch (error) {
        uni.hideLoading();
        uni.showToast({
          title: error?.message || '智能体权限检查失败',
          icon: 'none'
        });
      }
    },

    // 有权限时：打开确认弹窗（不直接调用API）
    startAgentUseFlow(paper, studentId) {
      const paperTitle = paper?.title ? `《${paper.title}》` : '当前论文';
      this.agentUseModal = {
        studentId: studentId || '',
        paperTitle,
        ready: true,
        paper,
        loading: false,
        taskId: '',
        version: '',
        error: '',
        alreadySubmitted: false
      };
      this.showAgentUseModal = true;
    },

    // 学生主动点击「提交审查」后才调用API
    async doSubmitAgentAudit() {
      const paper = this.agentUseModal.paper;
      if (!paper) return;

      // 进入 loading 状态
      this.agentUseModal.ready = false;
      this.agentUseModal.loading = true;
      this.agentUseModal.error = '';

      // 将 version 和 auditCacheKey 提到 try 外，让 catch 也能访问
      let version = '';
      let auditCacheKey = '';

      try {
        // 优先读取缓存的论文详情（含 version）
        const cacheKey = `paperDetail_${paper.id}`;
        const cachedDetail = uni.getStorageSync(cacheKey);

        if (cachedDetail && cachedDetail.version) {
          version = cachedDetail.version;
        } else {
          const detail = await getPaperDetail(paper.id);
          if (detail && detail.version) {
            version = detail.version;
            try { uni.setStorageSync(cacheKey, detail); } catch (e) {}
          } else {
            const rawVersion = paper.version || '1.0';
            version = rawVersion.startsWith('v') ? rawVersion : `v${rawVersion}`;
          }
        }

        this.$set(this.agentUseModal, 'version', version);
        auditCacheKey = `agentAudit_${paper.id}_${version}`;

        // 防止同一论文+版本重复提交
        const existingAudit = uni.getStorageSync(auditCacheKey);
        if (existingAudit && existingAudit.taskId) {
          this.agentUseModal = {
            ...this.agentUseModal,
            loading: false,
            taskId: existingAudit.taskId,
            error: '',
            alreadySubmitted: true
          };
          return;
        }

        // 调用 /api/v1/agent/audit 提交审核任务
        const auditRes = await callAgentAudit(paper.id, version);
        const taskId = auditRes?.task_id || auditRes?.data?.task_id || '';

        // 缓存提交记录
        try { uni.setStorageSync(auditCacheKey, { taskId, submittedAt: new Date().toISOString() }); } catch (e) {}

        this.agentUseModal = {
          ...this.agentUseModal,
          loading: false,
          taskId,
          error: '',
          verifiedByQuery: false
        };
      } catch (error) {
        const errMsg = error?.message || error?.detail || String(error || '');
        // 后端已知bug：无论成功失败均返回该报错信息
        const isMysqlWriteError = errMsg.includes('任务记录未成功写入 MySQL') ||
                                   errMsg.includes('任务记录未成功写入');

        if (isMysqlWriteError && version) {
          // 尝试通过 by-paper 接口验证是否实际提交成功
          try {
            const checkRes = await getAgentTaskByPaper(paper.id, version);
            const data = this.unwrapAgentPayload(checkRes);
            const taskId = data?.task_id ?? data?.id ?? null;

            if (taskId) {
              // 接口返回了 task_id → 实际提交成功，写入缓存
              try {
                if (auditCacheKey) {
                  uni.setStorageSync(auditCacheKey, { taskId: String(taskId), submittedAt: new Date().toISOString() });
                }
                uni.setStorageSync(`agentTaskInfo_${paper.id}_${version}`, { task_id: taskId, paper_id: paper.id, version });
              } catch (e) {}

              this.agentUseModal = {
                ...this.agentUseModal,
                loading: false,
                taskId: String(taskId),
                error: '',
                verifiedByQuery: true
              };
              return;
            }
          } catch (checkErr) {
            // by-paper 接口也异常，属于真正失败
          }

          // by-paper 没有返回 task_id → 真正提交失败
          this.agentUseModal = {
            ...this.agentUseModal,
            loading: false,
            error: '提交失败，后端服务异常，请稍后重试'
          };
        } else {
          this.agentUseModal = {
            ...this.agentUseModal,
            loading: false,
            error: errMsg || '智能体调用失败，请稍后重试'
          };
        }
      }
    },
    
    async getPaperList(showLoading = true) {
      const userInfo = uni.getStorageSync('userInfo') || {};
      const numericUserId = parseInt(userInfo.sub || 0, 10);
      
      if (!numericUserId) {
        this.loading = false;
        const loadedFromStorage = this.loadPapersFromLocalStorage();
        if (!loadedFromStorage) {
          this.paperList = JSON.parse(JSON.stringify(this.backupPaperList));
        }
        return;
      }
      
      // 先尝试从本地存储加载，快速显示
      const cachedData = uni.getStorageSync(`studentPapers_${numericUserId}`);
      if (cachedData && Array.isArray(cachedData) && cachedData.length > 0 && !showLoading) {
        this.paperList = this.formatPaperData(cachedData);
        this.applyStoredStatusToPaperList();
        this.loading = false;
      } else if (showLoading) {
        this.loading = true;
      }
      
      try {
        const res = await getPaperList({ owner_id: numericUserId });
        this.loading = false;
        this.paperListKey++;
        
        let paperData = null;
        if (res && res.code === 200 && res.data && Array.isArray(res.data)) {
          paperData = res.data;
        } else if (Array.isArray(res)) {
          paperData = res;
        }
        
        if (paperData && paperData.length > 0) {
          const myPapers = paperData.filter(p => !p.owner_id || p.owner_id === numericUserId);
          this.paperList = this.formatPaperData(myPapers.length > 0 ? myPapers : paperData);
          uni.setStorageSync(`studentPapers_${numericUserId}`, paperData);
          
          // 应用本地存储的论文状态
          this.applyStoredStatusToPaperList();
          
          // 异步获取所有论文的详情和状态历史（不阻塞界面）
          this.fetchAllPaperDetails(this.paperList);
          
          // 异步获取教师工号并更新显示（不阻塞界面）
          const userInfo = uni.getStorageSync('userInfo') || {};
          const studentUsername = userInfo.username || '';
          if (studentUsername) {
            this.fetchAndApplyTeacherUsername(studentUsername);
          }
        } else if (paperData !== null) {
          this.paperList = [];
        } else {
          const loadedFromStorage = this.loadPapersFromLocalStorage(numericUserId);
          if (!loadedFromStorage) {
            this.paperList = [];
          }
          this.applyStoredStatusToPaperList();
        }
      } catch (err) {
        console.error('获取论文列表失败:', err);
        this.loading = false;
        this.paperListKey++;
        const loadedFromStorage = this.loadPapersFromLocalStorage(numericUserId);
        if (!loadedFromStorage) {
          this.paperList = [];
        }
      }
    },
    loadPapersFromLocalStorage(userId) {
      try {
        // 优先读取当前用户专属的本地缓存
        const key = userId ? `studentPapers_${userId}` : 'studentPapers';
        const savedPapers = uni.getStorageSync(key);
        if (savedPapers && Array.isArray(savedPapers) && savedPapers.length > 0) {
          this.paperList = this.formatPaperData(savedPapers);
          // 应用本地存储的论文状态
          this.applyStoredStatusToPaperList();
          return true;
        }
        return false;
      } catch (error) {
        console.error('加载本地论文数据失败:', error);
        return false;
      }
    },
    formatPaperData(list) {
      return list.map(item => {
        const rawStatus = item.status || item.state || item.paper_status || '';
        const status = (rawStatus || '').toLowerCase().trim();
        const rawTrim = String(rawStatus).trim();
        const statusText = rawTrim ? studentDisplayStatusLabel(rawStatus) : '';
        console.log('[formatPaperData] 原始状态:', rawStatus, '-> 标准化:', status, '-> 展示:', statusText);
        const statusHistory = item.statusHistory || item.status_history || [
          {
            time: item.updateTime || item.updated_at || new Date().toLocaleString('zh-CN'),
            content: `当前状态：${studentDisplayStatusLabel(rawStatus)}`
          }
        ];
        
        // 尝试从多个字段获取标题
        let paperTitle = item.title || item.name || item.paper_name || item.filename || item.file_name;
        
        // 如果没有标题，尝试从 oss_key 或 fileUrl 中提取文件名
        if (!paperTitle && (item.oss_key || item.fileUrl)) {
          const path = item.oss_key || item.fileUrl;
          const fileName = path.split('/').pop(); // 获取路径最后一部分
          if (fileName) {
            // 移除时间戳前缀（如 20260310153710467633_）
            paperTitle = fileName.replace(/^\d+_/, '').replace(/\.[^/.]+$/, '');
          }
        }
        
        // 如果还是没有，使用默认标题
        if (!paperTitle) {
          paperTitle = '无标题论文';
        }
        
        return {
          id: item.id || '',
          title: paperTitle,
          fileName: item.filename || item.file_name || item.name || item.oss_key || paperTitle,
          oss_key: item.oss_key || '',
          pdf_oss_key: item.pdf_oss_key || '',
          version: String(item.version || item.latest_version || '1.0').replace(/^v/i, ''),
          teacher: item.teacher || (item.teacher_id ? `教师ID: ${item.teacher_id}` : '未分配指导教师'),
          _rawTeacherId: item.teacher_id || null, // 保留原始数字teacher_id，用于后续查询工号
          status: status, // 使用标准化后的状态值（小写），确保后续判断一致
          statusText: statusText,
          updateTime: item.updateTime || item.updated_at || item.created_at || '暂无更新时间',
          fileUrl: item.fileUrl || item.oss_key || item.file_url || '',
          statusHistory: statusHistory
        };
      });
    },
    /**
     * 从 oss_key 构建静态文件 URL
     */
    getFileUrl(rawPath) {
      if (!rawPath) return '';

      if (/^https?:\/\//i.test(rawPath)) {
        return rawPath;
      }

      if (rawPath.startsWith('/doc/essay/')) {
        return `${config.baseURL}${encodeURI(rawPath)}`;
      }

      const filename = rawPath.split('/').pop();
      if (!filename) return '';

      return `${config.baseURL}/doc/essay/${encodeURIComponent(filename)}`;
    },

    /**
     * 通过论文详情接口获取最新文件信息，并同步回列表
     */
    async fetchPaperDetailForAction(paper) {
      const detail = await getPaperDetail(paper.id);
      if (!detail?.id) {
        throw new Error('获取论文详情失败');
      }

      const normalizedStatus = (detail.status || '').toLowerCase().trim();
      const mergedPaper = {
        ...paper,
        oss_key: detail.oss_key || paper.oss_key || '',
        pdf_oss_key: detail.pdf_oss_key || paper.pdf_oss_key || '',
        version: String(detail.version || paper.version || '1.0').replace(/^v/i, ''),
        updateTime: detail.updated_at || paper.updateTime || '暂无更新时间',
        fileUrl: detail.oss_key || detail.pdf_oss_key || paper.fileUrl || ''
      };

      if (normalizedStatus) {
        mergedPaper.status = normalizedStatus;
        mergedPaper.statusText = this.formatStatusText(detail.status);
        mergedPaper.statusHistory = this.buildStatusHistory(detail);
      }

      const index = this.paperList.findIndex(item => item.id === paper.id);
      if (index !== -1) {
        Object.keys(mergedPaper).forEach(key => {
          this.$set(this.paperList[index], key, mergedPaper[key]);
        });
      }

      return mergedPaper;
    },

    /**
     * 预览论文 - 使用 WordPreview 组件
     */
    async viewPaper(paper) {
      console.log('[viewPaper] 开始预览论文:', paper);

      if (!paper.id) {
        uni.showToast({title: '论文信息不完整', icon: "none", duration:1500});
        return;
      }

      let previewWindow = null;
      // #ifdef H5
      previewWindow = window.open('', '_blank');
      if (previewWindow) {
        previewWindow.document.write('<title>论文预览加载中</title><p style="padding:24px;font-family:Arial,sans-serif;color:#333;">正在加载论文预览...</p>');
      }
      // #endif

      uni.showLoading({ title: '加载论文中...', mask: true });

      let paperDetail = paper;
      try {
        paperDetail = await this.fetchPaperDetailForAction(paper);
      } catch (error) {
        console.error('[viewPaper] 获取论文详情失败:', error);
        // #ifdef H5
        if (previewWindow) {
          previewWindow.close();
        }
        // #endif
        uni.hideLoading();
        uni.showToast({ title: error.message || '获取论文详情失败', icon: 'none', duration: 1800 });
        return;
      }

      const hasRealFile = paperDetail.oss_key || paperDetail.pdf_oss_key;
      console.log('[viewPaper] 是否有真实文件:', hasRealFile, '详情:', paperDetail);

      if (!hasRealFile) {
        // #ifdef H5
        if (previewWindow) {
          previewWindow.close();
        }
        // #endif
        uni.hideLoading();
        uni.showModal({
          title: '提示',
          content: '该论文暂无可预览文件，请稍后重试或下载源文件查看。',
          showCancel: false,
          confirmText: '知道了'
        });
        return;
      }

      // 预览优先使用 PDF 版本
      console.log('[viewPaper] pdf_oss_key:', paperDetail.pdf_oss_key);
      if (!paperDetail.pdf_oss_key) {
        // #ifdef H5
        if (previewWindow) {
          previewWindow.close();
        }
        // #endif
        uni.hideLoading();
        uni.showModal({
          title: '提示',
          content: '该论文暂无 PDF 版本，是否下载 Word 版本查看？',
          confirmText: '下载',
          cancelText: '取消',
          success: (res) => {
            if (res.confirm) this.downloadPaper(paperDetail);
          }
        });
        return;
      }

      const fileUrl = this.getFileUrl(paperDetail.pdf_oss_key);
      if (!fileUrl) {
        // #ifdef H5
        if (previewWindow) {
          previewWindow.close();
        }
        // #endif
        uni.hideLoading();
        uni.showToast({ title: '论文预览链接无效', icon: 'none', duration: 1500 });
        return;
      }
      console.log('[viewPaper] 预览PDF URL:', fileUrl);

      // #ifdef H5
      if (previewWindow) {
        previewWindow.location.href = fileUrl;
      } else {
        window.open(fileUrl, '_blank');
      }
      // #endif
      // #ifndef H5
      uni.downloadFile({
        url: fileUrl,
        success: (res) => {
          if (res.statusCode === 200) {
            uni.openDocument({filePath: res.tempFilePath, showMenu: true});
          }
        }
      });
      // #endif
      uni.hideLoading();
    },
    
    /**
     * 关闭 Word 预览
     */
    closeWordPreview() {
      this.showWordPreview = false;
      this.previewFileUrl = '';
      this.previewTitle = '';
      this.previewPaper = null;
    },
    
    /**
     * 处理预览中的下载请求
     */
    handlePreviewDownload({ url, title }) {
      // 保存当前论文引用
      const paper = this.previewPaper;
      
      // 先关闭预览组件，避免遮挡下载确认对话框
      this.showWordPreview = false;
      
      // 延迟执行下载，等待预览组件完全关闭
      setTimeout(() => {
        if (paper) {
          this.downloadPaper(paper);
        }
        // 清理预览数据
        this.previewFileUrl = '';
        this.previewTitle = '';
        this.previewPaper = null;
      }, 300);
    },
    
    /**
     * 处理预览错误
     */
    handlePreviewError({ error, message, url }) {
      console.log('预览错误:', message);
      
      // 如果预览失败，提示用户下载查看
      uni.showModal({
        title: '预览提示',
        content: message || '文档预览失败，是否下载查看？',
        confirmText: '下载查看',
        cancelText: '取消',
        success: (res) => {
          if (res.confirm && this.previewPaper) {
            this.downloadPaper(this.previewPaper);
          }
        }
      });
    },
    // 显示确认弹窗
    showConfirm(title, content, callback) {
      this.confirmModalTitle = title;
      this.confirmModalContent = content;
      this.confirmModalCallback = callback;
      this.showConfirmModal = true;
    },
    
    // 处理确认弹窗确认
    handleConfirmModalConfirm() {
      this.showConfirmModal = false;
      if (this.confirmModalCallback) {
        this.confirmModalCallback(true);
      }
    },
    
    // 处理确认弹窗取消
    handleConfirmModalCancel() {
      this.showConfirmModal = false;
      if (this.confirmModalCallback) {
        this.confirmModalCallback(false);
      }
    },
    
    async downloadPaper(paper) {
      console.log('[downloadPaper] 开始下载论文:', paper);
      
      if (!paper.id) {
        uni.showToast({title: '论文信息不完整', icon: "none"});
        return;
      }

      uni.showLoading({ title: '加载论文中...', mask: true });

      let paperDetail = paper;
      try {
        paperDetail = await this.fetchPaperDetailForAction(paper);
      } catch (error) {
        console.error('[downloadPaper] 获取论文详情失败:', error);
        uni.hideLoading();
        uni.showToast({ title: error.message || '获取论文详情失败', icon: 'none' });
        return;
      }

      uni.hideLoading();

      if (!paperDetail.oss_key) {
        console.log('[downloadPaper] 论文没有oss_key，详情:', paperDetail);
        this.showConfirm('提示', '该论文为演示数据，无法下载。请先上传真实论文文件。', (confirmed) => {
          // 只是提示，不需要处理确认结果
        });
        return;
      }

      // 下载使用 Word (docx) 版本
      const fileUrl = this.getFileUrl(paperDetail.oss_key);
      if (!fileUrl) {
        uni.showToast({ title: '论文下载链接无效', icon: 'none' });
        return;
      }
      const downloadName = `${paperDetail.title || '论文'}.docx`;
      
      console.log('[downloadPaper] 下载URL:', fileUrl, '文件名:', downloadName);

      this.showConfirm('下载确认', `即将下载论文《${paperDetail.title}》，是否继续？`, async (confirmed) => {
        if (!confirmed) {
          console.log('[downloadPaper] 用户取消下载');
          return;
        }

        uni.showLoading({title: '下载中...', mask: true});

        // #ifdef H5
        try {
          const response = await fetch(fileUrl);
          if (!response.ok) throw new Error('下载失败');
          const blob = await response.blob();
          const url = window.URL.createObjectURL(blob);
          const link = document.createElement('a');
          link.href = url;
          link.download = downloadName;
          document.body.appendChild(link);
          link.click();
          document.body.removeChild(link);
          window.URL.revokeObjectURL(url);
          uni.showToast({title: '下载完成', icon: "success"});
        } catch (error) {
          console.error('[downloadPaper] 下载失败:', error);
          uni.showToast({title: '下载失败', icon: "none"});
        } finally {
          uni.hideLoading();
        }
        // #endif

        // #ifndef H5
        uni.downloadFile({
          url: fileUrl,
          success: (downloadRes) => {
            if (downloadRes.statusCode === 200) {
              uni.saveFile({
                tempFilePath: downloadRes.tempFilePath,
                success: () => uni.showToast({title: '文件已保存', icon: "success"}),
                fail: () => uni.showToast({title: '保存失败', icon: "none"})
              });
            } else {
              uni.showToast({title: '下载失败', icon: "none"});
            }
            uni.hideLoading();
          },
          fail: () => {
            uni.showToast({title: '下载失败', icon: "none"});
            uni.hideLoading();
          }
        });
        // #endif
      });
    },
    
    // 删除论文关联的附件
    async deletePaperAttachments(paperId) {
      try {
        // 获取该论文的附件列表
        const { getAttachmentList, deleteAttachment } = await import('../../api/student.js');
        const res = await getAttachmentList(paperId);
        
        // 解析附件列表
        let attachments = [];
        if (res && res.code === 200 && Array.isArray(res.data)) {
          attachments = res.data;
        } else if (Array.isArray(res)) {
          attachments = res;
        }
        
        if (attachments.length === 0) {
          console.log(`论文 ${paperId} 没有关联附件`);
          return;
        }
        
        console.log(`论文 ${paperId} 关联 ${attachments.length} 个附件，开始删除...`);
        
        // 并行删除所有附件
        const deletePromises = attachments.map(attachment => {
          const attachmentId = attachment.id || attachment.material_id;
          if (attachmentId) {
            return deleteAttachment(attachmentId).catch(err => {
              console.error(`删除附件 ${attachmentId} 失败:`, err);
              // 单个附件删除失败不影响整体流程
              return null;
            });
          }
          return Promise.resolve(null);
        });
        
        await Promise.all(deletePromises);
        console.log(`论文 ${paperId} 的附件删除完成`);
      } catch (error) {
        console.error(`获取论文 ${paperId} 附件列表失败:`, error);
        // 获取附件列表失败不阻止论文删除
      }
    },
    
    deletePaper(paperId, paperTitle) {
      this.showConfirm('确认删除', `确定要删除论文《${paperTitle}》吗？`, async (confirmed) => {
        if (confirmed) {
          try {
            // 先获取该论文的附件列表并删除
            await this.deletePaperAttachments(paperId);
            
            // 调用后端 API 删除论文
            await deletePaper(paperId);
            
            // 从本地存储中删除
            const userInfo = uni.getStorageSync('userInfo') || {};
            const numericUserId = parseInt(userInfo.sub || 0, 10);
            const savedPapers = uni.getStorageSync(`studentPapers_${numericUserId}`);
            if (savedPapers && Array.isArray(savedPapers)) {
              const updatedPapers = savedPapers.filter(paper => paper.id !== paperId);
              uni.setStorageSync(`studentPapers_${numericUserId}`, updatedPapers);
            }
            
            // 从列表中移除
            this.paperList = this.paperList.filter(paper => paper.id !== paperId);
            uni.showToast({title: '删除成功', icon: "success"});
          } catch (error) {
            console.error('删除论文失败:', error);
            uni.showToast({title: '删除失败：' + (error.message || '未知错误'), icon: "none"});
          }
        }
      });
    },
    
    // 查看批注 - 跳转到论文预览页面并显示批注定位
    async viewAnnotations(paper) {
      uni.showLoading({title: '加载批注中...'});

      try {
        // 先获取批注数据
        const res = await getAnnotationsByPaperId(paper.id);
        uni.hideLoading();

        let annotations = [];
        if (res && Array.isArray(res)) {
          annotations = res.map(item => {
            const parsed = this.parseAnnotationContent(item.content);
            return {
              id: item.id,
              source: item.author_id === 1 ? 'teacher' : 'AI',
              content: parsed,
              rawContent: item.content,
              time: this.formatTime(item.created_at),
              coordinates: item.coordinates,
              paragraphId: item.paragraph_id,
              authorId: item.author_id,
              created_at: item.created_at,
              processed: false
            };
          });
        }

        if (annotations.length === 0) {
          uni.showToast({title: '暂无批注', icon: 'none'});
          return;
        }

        // 将批注数据存储到全局，供 paperPreview 使用
        const app = getApp();
        app.globalData.previewAnnotations = annotations;
        
        // 跳转到论文预览页面
        const url = `/pages/student/paperPreview?id=${paper.id}&title=${encodeURIComponent(paper.title || '论文预览')}&showAnnotations=true`;
        console.log('[跳转] 目标URL:', url);
        uni.navigateTo({
          url: url,
          fail: (err) => {
            console.error('[跳转失败]', err);
            uni.showToast({ title: '页面跳转失败', icon: 'none' });
          }
        });
      } catch (error) {
        uni.hideLoading();
        console.error('获取批注失败:', error);
        uni.showToast({title: '获取批注失败', icon: 'none'});
      }
    },

    // 解析批注内容
    parseAnnotationContent(content) {
      if (!content) return { selected: '', annotation: '', suggestion: '' };
          
      // 提取各部分
      const selectedMatch = content.match(/\u9009\u4e2d\u5185\u5bb9：([\s\S]*?)(?=\u6279\u6ce8：|$)/);
      const annotationMatch = content.match(/\u6279\u6ce8：([\s\S]*?)(?=\u5efa\u8bae：|$)/);
      const suggestionMatch = content.match(/\u5efa\u8bae：([\s\S]*)/);
          
      return {
        selected: selectedMatch ? selectedMatch[1].replace(/<[^>]+>/g, '').trim() : '',
        annotation: annotationMatch ? annotationMatch[1].replace(/<[^>]+>/g, '').trim() : '',
        suggestion: suggestionMatch ? suggestionMatch[1].replace(/<[^>]+>/g, '').trim() : ''
      };
    },

    // 格式化时间
    formatTime(isoTime) {
      if (!isoTime) return '';
      const date = new Date(isoTime);
      return date.toLocaleString('zh-CN', {
        year: 'numeric',
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit'
      });
    },
    
    // 关闭批注模态框
    closeAnnotationModal() {
      this.showAnnotationModal = false;
      this.currentPaper = null;
      this.annotations = [];
    }
  }
};
</script>

<style scoped>
/* ==================== CSS变量定义 - 与工作台保持一致 ==================== */
.paper-list-page {
  /* Primary Colors */
  --primary: #005bbf;
  --primary-container: #1a73e8;
  --primary-fixed: #d6e3ff;
  --on-primary-fixed: #002e6d;

  /* Surface Hierarchy */
  --surface: #f8f9fa;
  --surface-bright: #ffffff;
  --surface-container-low: #f3f4f5;
  --surface-container-lowest: #ffffff;
  --surface-container-high: #e7e8e9;
  --surface-variant: rgba(225, 227, 228, 0.6);

  /* Text Colors */
  --on-surface: #191c1d;
  --on-surface-variant: #5a5f61;
  --outline: #727785;
  --outline-variant: rgba(193, 198, 214, 0.15);

  /* Semantic Colors */
  --error: #ba1a1a;
  --error-container: #ffdad6;
  --on-error-container: #410002;
  --tertiary: #006a5f;
  --tertiary-container: #b2dfdb;
  --on-tertiary-container: #00201c;
  --secondary-container: #e0e2ec;
  --on-secondary-container: #191c1d;
  --amber-tint: #fef3c7;
  --on-amber: #92400e;

  /* Shadows - Ambient light physics */
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow: 0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
  --shadow-ambient: 0 16px 48px rgba(25, 28, 29, 0.08);
  --shadow-primary: 0 4px 16px rgba(0, 91, 191, 0.25);

  /* Typography */
  --font-display: 'Manrope', -apple-system, BlinkMacSystemFont, sans-serif;
  --font-body: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;

  /* Spacing */
  --spacing-1: 0.25rem;
  --spacing-2: 0.5rem;
  --spacing-3: 0.75rem;
  --spacing-4: 1rem;
  --spacing-5: 1.25rem;
  --spacing-6: 1.5rem;
  --spacing-8: 2rem;

  /* Border Radius */
  --radius-sm: 0.25rem;
  --radius-md: 0.75rem;
  --radius-lg: 1rem;
  --radius-full: 9999px;

  /* 动画缓动函数 */
  --ease-out-expo: cubic-bezier(0.16, 1, 0.3, 1);
  --ease-out-back: cubic-bezier(0.34, 1.56, 0.64, 1);
  --ease-in-out-smooth: cubic-bezier(0.4, 0, 0.2, 1);
  --ease-spring: cubic-bezier(0.68, -0.55, 0.265, 1.55);

  /* 动画时长 */
  --duration-fast: 150ms;
  --duration-normal: 300ms;
  --duration-slow: 500ms;
  --duration-enter: 600ms;
}

/* ==================== 全局性能优化 ==================== */
/* 减少动画对性能的影响 */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}

/* ==================== 顶部标题栏 ==================== */
.top-header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 100;
  height: 64px;
  background: var(--surface-container-lowest);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 24px;
  box-sizing: border-box;
  box-shadow: var(--shadow);
}

.header-left {
  display: flex;
  align-items: center;
}

.header-brand {
  display: flex;
  align-items: center;
  gap: 12px;
}

.header-brand-icon {
  width: 40px;
  height: 40px;
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-container) 100%);
  border-radius: var(--radius-md);
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  box-shadow: var(--shadow-primary);
}

.header-brand-icon .material-symbols-outlined {
  font-size: 20px;
}

.header-brand-text {
  display: flex;
  flex-direction: column;
}

.header-brand-title {
  font-size: 18px;
  font-weight: 800;
  font-family: var(--font-display);
  color: var(--primary);
  line-height: 1.2;
  letter-spacing: -0.01em;
}

.header-brand-subtitle {
  font-size: 12px;
  font-weight: 600;
  font-family: var(--font-body);
  color: var(--on-surface-variant);
  letter-spacing: 0.1em;
  text-transform: uppercase;
  margin-top: 2px;
}

.top-header .header-right {
  display: flex;
  align-items: center;
  gap: 16px;
}

.header-user-info {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 4px 16px 4px 4px;
  border-radius: var(--radius-full);
  background: var(--surface-container-low);
}

.profile-avatar {
  width: 36px;
  height: 36px;
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-container) 100%);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-weight: 600;
  font-size: 14px;
}

.profile-info {
  display: flex;
  flex-direction: column;
}

.profile-name {
  font-size: 16px;
  font-weight: 600;
  color: var(--on-surface);
  line-height: 1.2;
}

.profile-role {
  font-size: 12px;
  color: var(--on-surface-variant);
  line-height: 1.2;
}

/* 优化滚动性能 */
.paper-list-page {
  min-height: 100vh;
  background: var(--surface);
  font-family: var(--font-body);
  -webkit-overflow-scrolling: touch;
  transition: opacity 0.3s ease-out, transform 0.3s ease-out;
}

/* 页面淡出动画 */
.paper-list-page.page-fade-out {
  opacity: 0;
  transform: scale(0.98);
}

/* 启用GPU加速 */
.gpu-accelerated {
  transform: translateZ(0);
  will-change: transform;
}

/* ==================== 页面入场动画 ==================== */
.page-enter {
  animation: pageEnter 0.4s ease-out both;
}

@keyframes pageEnter {
  from {
    opacity: 0;
    transform: translateY(10rpx);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

/* ==================== 列表重新排序动画 ==================== */
.paper-list-move {
  transition: transform 0.5s cubic-bezier(0.16, 1, 0.3, 1);
}

.paper-list-enter-active,
.paper-list-leave-active {
  transition: all 0.4s cubic-bezier(0.16, 1, 0.3, 1);
}

.paper-list-enter {
  opacity: 0;
  transform: translateY(15rpx) scale(0.97);
}

.paper-list-leave-to {
  opacity: 0;
  transform: translateX(-15rpx);
}

/* 顶部标题栏 - 已移除，使用原生导航栏 */

/* 导航栏 */
.nav-bar {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 90rpx;
  background: var(--surface-bright);
  border-bottom: 1px solid var(--surface-container-high);
  display: flex;
  align-items: center;
  padding: 0 40rpx;
  box-shadow: var(--shadow);
  z-index: 99;
  overflow-x: auto;
  white-space: nowrap;
  box-sizing: border-box;
}

.nav-bar .nav-item {
  display: inline-flex;
  align-items: center;
  padding: 18rpx 30rpx;
  margin-right: 25rpx;
  border-radius: var(--radius-full);
  color: var(--on-surface-variant);
  font-size: 28rpx;
  font-weight: 500;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  cursor: pointer;
  white-space: nowrap;
  position: relative;
  background-color: var(--surface-container-low);
}

.nav-bar .nav-item:hover {
  background-color: var(--primary-fixed);
  color: var(--primary);
  transform: translateY(-2rpx);
  box-shadow: var(--shadow-sm);
}

.nav-bar .nav-item.active {
  background-color: var(--primary);
  color: #fff;
  box-shadow: var(--shadow-primary);
}

.nav-bar .nav-text {
  margin-right: 10rpx;
}

.nav-bar .nav-count {
  background-color: var(--primary);
  color: #fff;
  padding: 5rpx 15rpx;
  border-radius: var(--radius-full);
  font-size: 22rpx;
  font-weight: bold;
  min-width: 40rpx;
  text-align: center;
  box-shadow: var(--shadow-sm);
}

.nav-bar .nav-item.active .nav-count {
  background-color: #fff;
  color: var(--primary);
}

/* 主布局包裹层 */
.main-layout-wrapper {
  display: flex;
  justify-content: flex-start;
  width: 100%;
}

/* 主布局 */
.main-layout {
  display: flex;
  height: 100vh;
  background: var(--surface);
  width: 100%;
  max-width: none;
  justify-content: flex-start;
  gap: 24rpx;
  padding: 64px 24rpx 0 280px;
  box-sizing: border-box;
  overflow: hidden;
}

/* 左侧边栏 - 与工作台保持一致 256px */
.sidebar {
  position: fixed;
  left: 0;
  top: 64px;
  height: calc(100vh - 64px);
  width: 256px;
  background: var(--surface-bright);
  box-shadow: var(--shadow);
  padding: 24rpx;
  overflow-y: auto;
  box-sizing: border-box;
  z-index: 98;
  display: flex;
  flex-direction: column;
}

.sidebar-title {
  font-size: 34rpx;
  font-weight: 700;
  color: var(--on-surface);
  padding-bottom: 20rpx;
  border-bottom: 1px solid var(--surface-container-high);
  margin-bottom: 30rpx;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.sidebar-content {
  padding: 0;
}

.stats-panel {
  display: flex;
  flex-direction: column;
  gap: 20rpx;
}

.stat-item {
  display: flex;
  align-items: center;
  gap: 20rpx;
  padding: 28rpx 20rpx;
  background: var(--surface-container-low);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-sm);
  transition: all 0.3s ease;
  position: relative;
  overflow: hidden;
}

.stat-item::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 6rpx;
  height: 100%;
  background: var(--primary);
  border-radius: 6rpx 0 0 6rpx;
}

.stat-item:hover {
  transform: translateY(-2rpx);
  box-shadow: var(--shadow);
  background: var(--surface-bright);
  border-color: var(--primary);
}

/* ==================== 可点击状态项动画优化 ==================== */
.stat-item.clickable {
  cursor: pointer;
  position: relative;
  overflow: hidden;
}

/* 波纹效果 */
.stat-item.clickable::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 0;
  height: 0;
  background: var(--primary-fixed);
  border-radius: 50%;
  transform: translate(-50%, -50%);
  transition: width 0.6s cubic-bezier(0.16, 1, 0.3, 1), height 0.6s cubic-bezier(0.16, 1, 0.3, 1);
}

.stat-item.clickable:active::after {
  width: 300rpx;
  height: 300rpx;
}

.stat-item.clickable:active {
  transform: translateY(-1rpx) scale(0.98);
  box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.08);
  transition: all 0.15s cubic-bezier(0.16, 1, 0.3, 1);
}

/* ==================== 统计图标动画优化 ==================== */
.stat-icon {
  font-size: 44rpx;
  width: 80rpx;
  height: 80rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: var(--primary-fixed);
  border-radius: var(--radius-sm);
  color: var(--primary);
  flex-shrink: 0;
  transition: 
    transform 0.4s cubic-bezier(0.34, 1.56, 0.64, 1),
    background-color 0.3s ease,
    color 0.3s ease;
  font-family: 'Material Symbols Outlined', sans-serif;
  font-variation-settings: 'FILL' 0, 'wght' 400, 'GRAD' 0, 'opsz' 24;
}

.stat-item:hover .stat-icon {
  transform: scale(1.15) rotate(5deg);
  background-color: var(--primary);
  color: #fff;
}

.stat-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 5rpx;
}

.stat-label {
  font-size: 26rpx;
  color: var(--on-surface-variant);
  font-weight: 500;
  text-align: center;
}

.stat-value {
  font-size: 48rpx;
  font-weight: 800;
  color: var(--primary);
  line-height: 1.2;
}

.stat-item:hover .stat-value {
  color: var(--primary);
}

.agent-stat-item .stat-icon {
  background-color: rgba(8, 151, 156, 0.12);
  color: #08979c;
}

.agent-stat-item:hover .stat-icon {
  background-color: #08979c;
  color: #fff;
}

.agent-stat-title {
  font-size: 28rpx;
  color: var(--on-surface);
  font-weight: 700;
  line-height: 1.35;
}

.agent-stat-desc {
  font-size: 22rpx;
  color: var(--on-surface-variant);
  line-height: 1.5;
}

.agent-stat-item.disabled {
  opacity: 0.7;
  cursor: not-allowed;
}

.agent-stat-item.disabled:hover {
  transform: none;
  box-shadow: var(--shadow-sm);
  background: var(--surface-container-low);
}

.agent-stat-item.disabled:hover .stat-icon {
  transform: none;
  background-color: rgba(8, 151, 156, 0.12);
  color: #08979c;
}

.agent-permission-entry .stat-icon {
  background-color: rgba(245, 158, 11, 0.12);
  color: #b45309;
}

.agent-permission-entry:hover .stat-icon {
  background-color: #d97706;
  color: #fff;
}

.agent-report-entry .stat-icon {
  background-color: rgba(99, 102, 241, 0.12);
  color: #4f46e5;
}

.agent-report-entry:hover .stat-icon {
  background-color: #4f46e5;
  color: #fff;
}

/* ==================== 侧边栏菜单动画优化 ==================== */
.sidebar-menu {
  margin-top: 30rpx;
  padding-top: 20rpx;
  border-top: 1px solid var(--surface-container-high);
  display: flex;
  flex-direction: column;
  gap: 20rpx;
  position: relative;
  z-index: 10;
}

.menu-item {
  display: flex;
  align-items: center;
  gap: 20rpx;
  padding: 25rpx;
  background-color: var(--surface-container-low);
  border-radius: var(--radius-md);
  border: 1px solid var(--surface-container-high);
  transition: 
    transform 0.4s cubic-bezier(0.34, 1.56, 0.64, 1),
    box-shadow 0.3s cubic-bezier(0.16, 1, 0.3, 1),
    background-color 0.3s ease,
    border-color 0.3s ease;
  cursor: pointer;
  position: relative;
  overflow: hidden;
  pointer-events: auto;
}

.menu-item::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 6rpx;
  height: 100%;
  background: var(--primary);
  border-radius: 6rpx 0 0 6rpx;
  transition: width 0.3s cubic-bezier(0.16, 1, 0.3, 1);
}

.menu-item:hover {
  transform: translateY(-4rpx) scale(1.02);
  box-shadow: var(--shadow);
  background-color: var(--surface-bright);
  border-color: var(--primary);
}

.menu-item:hover::before {
  width: 8rpx;
}

.menu-item:active {
  transform: translateY(-1rpx) scale(0.98);
  transition: all 0.1s cubic-bezier(0.16, 1, 0.3, 1);
}

.menu-icon {
  font-size: 40rpx;
  width: 70rpx;
  height: 70rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: var(--primary-fixed);
  border-radius: var(--radius-sm);
  transition: all 0.3s;
  font-family: 'Material Symbols Outlined', sans-serif;
  font-variation-settings: 'FILL' 0, 'wght' 400, 'GRAD' 0, 'opsz' 24;
}

.menu-item:hover .menu-icon {
  background-color: var(--primary);
  transform: scale(1.1);
}

.menu-text {
  flex: 1;
  font-size: 30rpx;
  font-weight: 500;
  color: var(--on-surface);
  transition: color 0.3s;
}

.menu-item:hover .menu-text {
  color: var(--primary);
}

/* 右侧内容区 - 扩展至全屏，支持滚动 */
.content-area {
  flex: 1;
  max-width: none;
  padding: 16px 30rpx 30rpx;
  background: var(--surface);
  overflow-y: auto;
  box-sizing: border-box;
  min-height: 0;
}

.content-wrapper {
  max-width: none;
  margin: 0;
  width: 100%;
}

/* 加载中样式 */
.loading-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 120rpx 0;
  background: var(--surface-bright);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow);
}

/* ==================== 加载动画优化 ==================== */
.loading-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 120rpx 0;
  background: var(--surface-bright);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow);
  animation: fadeIn 0.4s cubic-bezier(0.16, 1, 0.3, 1);
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10rpx); }
  to { opacity: 1; transform: translateY(0); }
}

.loading-spinner {
  width: 60rpx;
  height: 60rpx;
  position: relative;
  margin-bottom: 30rpx;
}

.loading-spinner::before,
.loading-spinner::after {
  content: '';
  position: absolute;
  border-radius: 50%;
  animation: pulse-ring 1.5s cubic-bezier(0.16, 1, 0.3, 1) infinite;
}

.loading-spinner::before {
  width: 100%;
  height: 100%;
  background: var(--primary);
  animation: pulse-ring 1.5s cubic-bezier(0.16, 1, 0.3, 1) infinite;
}

.loading-spinner::after {
  width: 70%;
  height: 70%;
  top: 15%;
  left: 15%;
  background: #fff;
  animation: pulse-dot 1.5s cubic-bezier(0.16, 1, 0.3, 1) infinite;
}

@keyframes pulse-ring {
  0% { transform: scale(0.8); opacity: 1; }
  50% { transform: scale(1.1); opacity: 0.7; }
  100% { transform: scale(0.8); opacity: 1; }
}

@keyframes pulse-dot {
  0% { transform: scale(0.6); }
  50% { transform: scale(1); }
  100% { transform: scale(0.6); }
}

/* 备用旋转加载动画 */
.loading-spinner-fallback {
  width: 60rpx;
  height: 60rpx;
  border: 4rpx solid #f0f0f0;
  border-top-color: #1890ff;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.loading-text {
  font-size: 28rpx;
  color: var(--on-surface-variant);
  animation: textPulse 1.5s ease-in-out infinite;
}

@keyframes textPulse {
  0%, 100% { opacity: 0.6; }
  50% { opacity: 1; }
}

.loading-text {
  font-size: 28rpx;
  color: var(--on-surface-variant);
}

/* 论文列表 */
.paper-list {
  display: flex;
  flex-direction: column;
  gap: 25rpx;
  position: relative;
}

/* ==================== 排序过渡动画 ==================== */
.paper-list-move {
  transition: transform 0.5s cubic-bezier(0.25, 0.46, 0.45, 0.94);
}

.paper-list-enter-active {
  transition: all 0.4s ease-out;
}

.paper-list-leave-active {
  transition: all 0.3s ease-in;
  position: absolute;
}

.paper-list-enter {
  opacity: 0;
  transform: translateY(30rpx);
}

.paper-list-leave-to {
  opacity: 0;
  transform: translateY(-30rpx);
}

/* ==================== 论文卡片样式 ==================== */
.paper-card {
  cursor: pointer;
  background: var(--surface-bright);
  border-radius: var(--radius-lg);
  padding: 35rpx;
  box-shadow: var(--shadow);
  border: 1rpx solid var(--surface-container-high);
  transition: 
    transform 0.4s cubic-bezier(0.34, 1.56, 0.64, 1),
    box-shadow 0.4s cubic-bezier(0.16, 1, 0.3, 1),
    border-color 0.3s ease;
  position: relative;
  will-change: transform, box-shadow;
  margin: 0 25rpx;
}

.paper-card:hover {
  transform: translateY(-6rpx) scale(1.01);
  box-shadow: var(--shadow-lg);
  border-color: var(--primary-fixed);
}

.paper-card:active {
  transform: translateY(-2rpx) scale(0.99);
  transition: all 0.15s cubic-bezier(0.16, 1, 0.3, 1);
}

/* 高亮样式（从工作台跳转过来） */
.paper-card.highlighted {
  border: 2px solid var(--primary);
  box-shadow: 0 0 20px rgba(0, 91, 191, 0.4), var(--shadow-lg);
  animation: highlightPulse 2s ease-in-out;
}

@keyframes highlightPulse {
  0%, 100% {
    box-shadow: 0 0 20px rgba(0, 91, 191, 0.4), var(--shadow-lg);
  }
  50% {
    box-shadow: 0 0 30px rgba(0, 91, 191, 0.6), var(--shadow-lg);
  }
}

/* 论文头部 */
.paper-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 25rpx;
  padding-bottom: 25rpx;
  border-bottom: 1px solid var(--surface-container-high);
}

/* 头部右侧区域 */
.paper-header-actions {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  justify-content: center;
  align-self: stretch;
  gap: 12rpx;
  flex-shrink: 0;
}

.paper-content-wrapper {
  overflow: hidden;
}

/* ==================== 获取状态按钮动画优化 ==================== */
.refresh-status-btn {
  height: 56rpx;
  padding: 0 24rpx;
  border-radius: var(--radius-full);
  background: var(--primary);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: 
    transform 0.3s cubic-bezier(0.34, 1.56, 0.64, 1),
    box-shadow 0.3s cubic-bezier(0.16, 1, 0.3, 1);
  box-shadow: var(--shadow-primary);
  flex-shrink: 0;
  position: relative;
  overflow: hidden;
}

.refresh-status-btn::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 0;
  height: 0;
  background: rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  transform: translate(-50%, -50%);
  transition: width 0.6s cubic-bezier(0.16, 1, 0.3, 1), height 0.6s cubic-bezier(0.16, 1, 0.3, 1);
}

.refresh-status-btn:active::before {
  width: 200rpx;
  height: 200rpx;
}

.refresh-status-btn:hover {
  transform: translateY(-3rpx) scale(1.02);
  box-shadow: 0 8rpx 20rpx rgba(0, 91, 191, 0.4);
}

.refresh-status-btn:active {
  transform: translateY(0) scale(0.96);
  box-shadow: var(--shadow-primary);
  transition: all 0.1s cubic-bezier(0.16, 1, 0.3, 1);
}

.refresh-status-btn.loading {
  animation: btnPulse 1.5s ease-in-out infinite;
}

@keyframes btnPulse {
  0%, 100% { box-shadow: var(--shadow-primary); }
  50% { box-shadow: 0 4rpx 20rpx rgba(0, 91, 191, 0.5); }
}

.refresh-text {
  font-size: 24rpx;
  color: #ffffff;
  font-weight: 500;
  white-space: nowrap;
}

.paper-info {
  flex: 1;
  min-width: 0;
}

.paper-title {
  font-size: 34rpx;
  font-weight: 700;
  color: var(--on-surface);
  display: block;
  margin-bottom: 12rpx;
  line-height: 1.4;
}

.paper-preview-hint {
  font-size: 24rpx;
  font-weight: 400;
  color: var(--primary);
  margin-left: 8rpx;
  opacity: 0.8;
}

.paper-meta {
  font-size: 26rpx;
  color: var(--on-surface-variant);
  display: block;
}

/* 状态徽章 - 统一蓝色 */
.status-badge {
  padding: 10rpx 24rpx;
  border-radius: var(--radius-full);
  font-size: 24rpx;
  font-weight: 500;
  white-space: nowrap;
  flex-shrink: 0;
  background-color: var(--primary-fixed);
  color: var(--primary);
  border: 1px solid var(--primary-fixed);
}

/* 论文内容 */
.paper-content {
  margin-bottom: 25rpx;
}

.paper-detail {
  display: flex;
  align-items: center;
  margin-bottom: 20rpx;
}

.detail-label {
  font-size: 26rpx;
  color: var(--on-surface-variant);
  margin-right: 10rpx;
}

.detail-value {
  font-size: 26rpx;
  color: var(--on-surface);
}

/* 进度条 */
.progress-section {
  margin-bottom: 25rpx;
  padding: 20rpx;
  background-color: var(--surface-container-low);
  border-radius: var(--radius-md);
}

.progress-track {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.progress-step {
  display: flex;
  flex-direction: column;
  align-items: center;
  flex: 1;
  position: relative;
}

.step-circle {
  width: 50rpx;
  height: 50rpx;
  border-radius: 50%;
  background-color: var(--surface-container-high);
  color: var(--on-surface-variant);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24rpx;
  font-weight: bold;
  margin-bottom: 10rpx;
  transition: all 0.3s;
  border: 2rpx solid var(--surface-container-high);
}

.progress-step.active .step-circle,
.progress-step.completed .step-circle {
  background-color: var(--primary);
  color: #fff;
  border-color: var(--primary);
  box-shadow: var(--shadow-primary);
}

.step-text {
  font-size: 20rpx;
  color: var(--on-surface-variant);
  text-align: center;
  transition: all 0.3s;
}

.progress-step.active .step-text,
.progress-step.completed .step-text {
  color: var(--primary);
  font-weight: 500;
}

.progress-line {
  flex: 1;
  height: 3rpx;
  background-color: var(--surface-container-high);
  margin: 0 10rpx;
  margin-bottom: 35rpx;
  transition: all 0.3s;
}

.progress-line.active {
  background-color: var(--primary);
}

/* 状态进度条动画效果 */
.step-circle {
  position: relative;
  overflow: visible;
}

.step-number {
  position: relative;
  z-index: 2;
}

/* 连接线流动动画 */
.progress-line {
  position: relative;
  overflow: hidden;
}

.line-flow {
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.8), transparent);
  animation: flow-move 2s linear infinite;
}

@keyframes flow-move {
  0% {
    left: -100%;
  }
  100% {
    left: 100%;
  }
}

/* 通知区域 - 统一蓝色 */
.notice-section {
  background-color: #e6f7ff;
  border-radius: 12rpx;
  padding: 20rpx;
  border: 1px solid #91d5ff;
}

.notice-header {
  margin-bottom: 15rpx;
}

.notice-title {
  font-size: 28rpx;
  font-weight: bold;
  color: #1890ff;
}

.notice-list {
  display: flex;
  flex-direction: column;
  gap: 12rpx;
}

.notice-item {
  background-color: #fff;
  border-radius: 8rpx;
  padding: 15rpx;
  border-left: 4rpx solid #1890ff;
}

.notice-time {
  font-size: 22rpx;
  color: #999;
  margin-bottom: 8rpx;
}

.notice-content {
  font-size: 26rpx;
  color: #333;
  line-height: 1.5;
}

/* 操作按钮 - 统一蓝色 */
.paper-actions {
  display: flex;
  gap: 16rpx;
  padding-top: 22rpx;
  border-top: 1px solid var(--surface-container-high);
}

.action-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8rpx;
  padding: 14rpx 20rpx;
  border-radius: var(--radius-sm);
  font-size: 24rpx;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s;
  border: none;
  flex: 1;
  background-color: var(--primary-fixed);
  color: var(--primary);
  border: 1px solid var(--primary-fixed);
  min-width: 0;
}

/* 删除按钮 - 浅红色背景 */
.action-btn.delete-btn {
  background-color: #ffebee;
  color: #e53e3e;
  border: 1px solid #ffcdd2;
}

.action-btn.delete-btn:hover {
  background-color: #ef5350;
  color: #fff;
  border-color: #ef5350;
}

.action-btn .btn-icon {
  font-size: 26rpx;
  font-family: 'Material Symbols Outlined', sans-serif;
  font-variation-settings: 'FILL' 0, 'wght' 400, 'GRAD' 0, 'opsz' 20;
}

.action-btn:hover {
  background-color: var(--primary);
  color: #fff;
  transform: translateY(-2rpx);
  box-shadow: var(--shadow-primary);
}

/* 更新按钮禁用状态 - 变灰不可用 */
.action-btn.update-btn.disabled {
  background-color: var(--surface-container-low);
  color: var(--on-surface-variant);
  border-color: var(--surface-container-high);
  cursor: not-allowed;
  opacity: 0.7;
}

.action-btn.update-btn.disabled:hover {
  background-color: var(--surface-container-low);
  color: var(--on-surface-variant);
  border-color: var(--surface-container-high);
  transform: none;
  box-shadow: none;
}

/* 查看审阅按钮样式 */
.action-btn.review-btn {
  background-color: #f6ffed;
  color: #52c41a;
  border: 1px solid #b7eb8f;
}

.action-btn.review-btn:hover {
  background-color: #52c41a;
  color: #fff;
  border-color: #52c41a;
}

/* 查看审阅按钮禁用状态 */
.action-btn.review-btn.disabled {
  background-color: var(--surface-container-low);
  color: var(--on-surface-variant);
  border-color: var(--surface-container-high);
  cursor: not-allowed;
  opacity: 0.7;
}

.action-btn.review-btn.disabled:hover {
  background-color: var(--surface-container-low);
  color: var(--on-surface-variant);
  border-color: var(--surface-container-high);
  transform: none;
  box-shadow: none;
}

/* ==================== 空状态动画优化 ==================== */
.empty-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 120rpx 40rpx;
  background: var(--surface-bright);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow);
  animation: emptyEnter 0.6s cubic-bezier(0.16, 1, 0.3, 1);
}

@keyframes emptyEnter {
  0% {
    opacity: 0;
    transform: translateY(30rpx) scale(0.95);
  }
  100% {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

.empty-icon {
  font-size: 80rpx;
  margin-bottom: 30rpx;
  animation: emptyFloat 3s ease-in-out infinite;
  font-family: 'Material Symbols Outlined', sans-serif;
  font-variation-settings: 'FILL' 0, 'wght' 400, 'GRAD' 0, 'opsz' 48;
}

@keyframes emptyFloat {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-10rpx); }
}

.empty-text {
  font-size: 32rpx;
  font-weight: bold;
  color: var(--on-surface);
  margin-bottom: 15rpx;
  animation: fadeInUp 0.5s cubic-bezier(0.16, 1, 0.3, 1) 0.2s both;
}

.empty-subtext {
  font-size: 26rpx;
  color: var(--on-surface-variant);
  animation: fadeInUp 0.5s cubic-bezier(0.16, 1, 0.3, 1) 0.3s both;
}

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(10rpx);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* ==================== 模态框动画优化 - 与工作台保持一致 ==================== */
/* 统一弹窗背景样式 */
.annotation-modal,
.update-modal,
.review-modal {
  position: fixed !important;
  top: 0 !important;
  left: 0 !important;
  right: 0 !important;
  bottom: 0 !important;
  background: rgba(0, 0, 0, 0.6) !important;
  display: flex !important;
  align-items: center !important;
  justify-content: center !important;
  z-index: 1000 !important;
}

@keyframes modalBackdropIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

/* 统一弹窗内容样式 */
.annotation-modal-content,
.update-modal-content,
.review-modal-content {
  background: #ffffff !important;
  border-radius: 12px;
  max-height: 80vh;
  overflow: hidden;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
  animation: modalSlideIn 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  display: flex;
  flex-direction: column;
}

@keyframes modalSlideIn {
  from {
    opacity: 0;
    transform: translateY(-30px) scale(0.95);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

/* 批注弹窗特殊尺寸 */
.annotation-modal-content {
  width: 90%;
  max-width: 900px;
  height: 80vh;
}

/* 更新弹窗特殊尺寸 */
.update-modal-content {
  width: 90%;
  max-width: 400px;
}

/* 审阅弹窗特殊尺寸 */
.review-modal-content {
  width: 90%;
  max-width: 500px;
}

/* 统一弹窗头部样式 - 与页面顶部标题栏一致 */
.annotation-modal-header,
.update-modal-header,
.review-modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 64px;
  padding: 0 24px;
  background: #ffffff;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
  border-radius: 12px 12px 0 0;
  flex-shrink: 0;
  box-sizing: border-box;
}

.annotation-header-left,
.annotation-header-right {
  display: flex;
  align-items: center;
  min-width: 40px;
}

.annotation-header-right {
  justify-content: flex-end;
}

/* 统一弹窗标题样式 - 与页面顶部标题栏一致 */
.annotation-modal-title,
.update-modal-title,
.review-modal-title {
  font-size: 16px;
  font-weight: 800;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  color: #005bbf;
  line-height: 1.2;
  letter-spacing: -0.01em;
  text-align: center;
  flex: 1;
}

/* 统一关闭按钮样式 - 与页面风格一致 */
.annotation-modal-close,
.update-modal-close,
.review-modal-close {
  font-size: 20px;
  color: #5f6368;
  cursor: pointer;
  padding: 8px;
  transition: color 0.2s;
  width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 8px;
  background: transparent;
}

.annotation-modal-close:hover,
.update-modal-close:hover,
.review-modal-close:hover {
  color: #191c1d;
  background: #f3f4f5;
}

/* 统一弹窗主体样式 */
.annotation-modal-body,
.update-modal-body,
.review-modal-body {
  padding: 20px;
  flex: 1;
  overflow-y: auto;
  background: #ffffff;
}

.annotation-paper-info {
  background: #f3f4f5;
  border-radius: 8px;
  padding: 16px;
  margin-bottom: 16px;
  border-left: 3px solid #005bbf;
}

.annotation-paper-title {
  font-size: 1rem;
  font-weight: 600;
  color: #191c1d;
  display: block;
  margin-bottom: 8px;
}

.annotation-paper-meta {
  font-size: 0.875rem;
  color: #5f6368;
  display: block;
}

.annotation-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
  max-height: calc(80vh - 200px);
  overflow-y: auto;
}

.annotation-item {
  background: #ffffff;
  border-radius: 8px;
  padding: 16px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
  border: 1px solid #e5e7eb;
  transition: all 0.3s;
}

.annotation-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.12);
}

.annotation-item.processed {
  border-left: 3px solid #38a169;
  background: linear-gradient(135deg, #f0fff4 0%, #ffffff 100%);
}

.annotation-item.unprocessed {
  border-left: 3px solid #005bbf;
  background: linear-gradient(135deg, #e8f4fd 0%, #ffffff 100%);
}

.annotation-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.annotation-source {
  display: flex;
  align-items: center;
  gap: 8px;
}

.source-icon {
  font-size: 1.25rem;
  font-family: 'Material Symbols Outlined', sans-serif;
  font-variation-settings: 'FILL' 0, 'wght' 400, 'GRAD' 0, 'opsz' 24;
}

.source-text {
  font-size: 0.875rem;
  font-weight: 600;
  color: #191c1d;
}

.annotation-status {
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 0.75rem;
  font-weight: 500;
}

.annotation-status.status-processed {
  background: #f0fff4;
  color: #1c4532;
}

.annotation-status.status-unprocessed {
  background: #e8f4fd;
  color: #1a3c6e;
}

/* 批注分区样式 */
.annotation-section {
  margin-bottom: 12px;
}

.annotation-section:last-of-type {
  margin-bottom: 0;
}

.section-title {
  font-size: 0.75rem;
  font-weight: 600;
  color: #5f6368;
  margin-bottom: 8px;
  padding-left: 8px;
  border-left: 2px solid #005bbf;
}

.section-content {
  font-size: 0.875rem;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  line-height: 1.6;
  padding: 12px;
  border-radius: 8px;
  background: #f3f4f5;
  max-height: 120px;
  overflow-y: auto;
  word-break: break-all;
}

.selected-content {
  color: #191c1d;
  background: #e8f4fd;
}

.annotation-content-text {
  color: #191c1d;
  background: #fffbeb;
}

.suggestion-content {
  color: #1c4532;
  background: #f0fff4;
}

.annotation-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.annotation-time {
  font-size: 0.75rem;
  color: #5f6368;
}

.annotation-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 32px 16px;
  text-align: center;
}

.annotation-empty .empty-icon {
  font-size: 3rem;
  margin-bottom: 16px;
  opacity: 0.5;
  font-family: 'Material Symbols Outlined', sans-serif;
  font-variation-settings: 'FILL' 0, 'wght' 400, 'GRAD' 0, 'opsz' 48;
}

.annotation-empty .empty-text {
  font-size: 1rem;
  font-weight: 500;
  color: #191c1d;
  margin-bottom: 8px;
}

.annotation-empty .empty-subtext {
  font-size: 0.875rem;
  color: #5f6368;
}

/* ========== 审阅详情弹窗样式 - 与工作台保持一致 ========== */
.review-modal {
  z-index: 1000;
}

/* review-modal-content、header、title、close、body 已在统一样式中定义 */

.review-paper-info {
  margin-bottom: 16px;
  padding-bottom: 12px;
  border-bottom: 1px solid #e5e7eb;
}

.review-paper-title {
  font-size: 1rem;
  font-weight: 600;
  color: #191c1d;
  display: block;
  margin-bottom: 8px;
}

.review-paper-meta {
  font-size: 0.875rem;
  color: #5f6368;
  display: block;
}

.review-content-section {
  margin-bottom: 16px;
}

.review-section-title {
  font-size: 0.875rem;
  font-weight: 600;
  color: #005bbf;
  margin-bottom: 12px;
  padding-left: 12px;
  border-left: 3px solid #005bbf;
}

.review-content-text {
  font-size: 0.875rem;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  color: #191c1d;
  line-height: 1.8;
  padding: 16px;
  background: #f3f4f5;
  border-radius: 8px;
  white-space: pre-wrap;
  word-break: break-all;
  max-height: 40vh;
  overflow-y: auto;
}

.review-time-section {
  display: flex;
  align-items: center;
  margin-top: 16px;
  padding-top: 12px;
  border-top: 1px solid #e5e7eb;
}

.review-time-label {
  font-size: 0.875rem;
  color: #5f6368;
  margin-right: 8px;
}

.review-time-value {
  font-size: 0.875rem;
  color: #191c1d;
  font-weight: 500;
}

.review-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: var(--spacing-8) var(--spacing-4);
  text-align: center;
}

.review-empty .empty-icon {
  font-size: 3rem;
  margin-bottom: var(--spacing-4);
  opacity: 0.5;
  font-family: 'Material Symbols Outlined', sans-serif;
  font-variation-settings: 'FILL' 0, 'wght' 400, 'GRAD' 0, 'opsz' 48;
}

.review-empty .empty-text {
  font-size: 1rem;
  font-weight: 500;
  color: var(--on-surface);
}

/* ========== 更新论文弹窗样式 - 与工作台保持一致 ========== */
.update-modal {
  z-index: 1000;
}

.update-paper-info {
  background: #f3f4f5;
  border-radius: 8px;
  padding: 16px;
  margin-bottom: 16px;
}

.update-paper-title {
  font-size: 1rem;
  font-weight: 600;
  color: #191c1d;
  display: block;
  margin-bottom: 8px;
}

.update-paper-current-version {
  font-size: 0.875rem;
  color: #5f6368;
}

.update-modal-tips {
  background: #e8f4fd;
  border-radius: 8px;
  padding: 12px 16px;
  margin-bottom: 16px;
}

.update-modal-tips text {
  display: block;
  font-size: 0.875rem;
  color: #1a3c6e;
  line-height: 1.6;
}

.version-input-section {
  margin-bottom: 16px;
}

.input-label {
  display: block;
  font-size: 0.875rem;
  font-weight: 500;
  color: #191c1d;
  margin-bottom: 8px;
}

.version-input {
  width: 100%;
  height: 44px;
  border: none;
  border-radius: 8px;
  padding: 0 16px;
  font-size: 0.875rem;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  color: #191c1d;
  background: #f3f4f5;
  box-sizing: border-box;
  transition: all 0.2s;
}

.version-input:focus {
  background: #e5e7eb;
  outline: none;
}

.version-hint {
  display: block;
  font-size: 0.75rem;
  color: #5f6368;
  margin-top: 8px;
}

.update-modal-btn {
  width: 100%;
  height: 44px;
  background: #f3f4f5;
  border: 2rpx dashed #e5e7eb;
  border-radius: 8px;
  font-size: 0.875rem;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  color: #191c1d;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 12px;
  cursor: pointer;
  transition: all 0.2s;
}

.update-modal-btn:active {
  background: #e5e7eb;
}

.update-modal-file-info {
  background: #e8f4fd;
  border-radius: 8px;
  padding: 12px 16px;
  margin-bottom: 16px;
}

.update-modal-file-name {
  font-size: 0.875rem;
  font-weight: 500;
  color: #1a3c6e;
  display: block;
  margin-bottom: 4px;
  word-break: break-all;
}

.update-modal-file-size {
  font-size: 0.75rem;
  color: #1a3c6e;
  opacity: 0.7;
}

.update-modal-submit-btn {
  width: 100%;
  height: 44px;
  background: #005bbf;
  border-radius: 8px;
  font-size: 0.875rem;
  color: white;
  font-weight: 500;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  display: flex;
  align-items: center;
  justify-content: center;
  border: none;
  cursor: pointer;
  transition: all 0.2s;
}

.update-modal-submit-btn:disabled {
  background: #e5e7eb;
  color: #5f6368;
  cursor: not-allowed;
}

.update-modal-submit-btn:not(:disabled):active {
  opacity: 0.9;
}

/* ==================== 响应式设计 ==================== */

/* 大屏幕 (>1400px) */
@media screen and (min-width: 1400px) {
  .main-layout {
    padding-left: 256px;
  }
  
  .sidebar {
    width: 256px;
  }
  
  .paper-card {
    padding: 32rpx;
  }
}

/* 中等屏幕 (992px-1399px) */
@media screen and (max-width: 1399px) and (min-width: 992px) {
  .main-layout {
    padding-left: 256px;
  }
  
  .sidebar {
    width: 256px;
  }
  
  .paper-card {
    padding: 28rpx;
  }
}

/* 平板屏幕 (768px-991px) */
@media screen and (max-width: 991px) and (min-width: 768px) {
  .main-layout {
    padding-left: 256px;
    padding-right: 16rpx;
  }
  
  .sidebar {
    width: 256px;
    padding: 20rpx 12rpx;
  }
  
  .sidebar-title {
    font-size: 30rpx;
  }
  
  .stat-label {
    font-size: 24rpx;
  }
  
  .stat-value {
    font-size: 40rpx;
  }
  
  .paper-card {
    padding: 24rpx;
  }
  
  .paper-title {
    font-size: 30rpx;
  }
  
  .header-brand-title {
    font-size: 16px;
  }
  
  .header-brand-subtitle {
    font-size: 10px;
  }
}

/* 小屏幕手机 (<768px) */
@media screen and (max-width: 767px) {
  /* 侧边栏变为顶部导航 */
  .sidebar {
    position: fixed;
    top: 64px;
    left: 0;
    right: 0;
    width: 100%;
    height: auto;
    max-height: 200px;
    padding: 16rpx;
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    gap: 16rpx;
    overflow-x: auto;
    overflow-y: hidden;
    z-index: 99;
    box-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.1);
  }
  
  .sidebar-title {
    width: 100%;
    font-size: 28rpx;
    margin-bottom: 12rpx;
    padding-bottom: 12rpx;
  }
  
  .stats-panel {
    flex-direction: row;
    flex-wrap: wrap;
    gap: 12rpx;
  }
  
  .stat-item {
    flex: 1;
    min-width: 140rpx;
    padding: 20rpx 16rpx;
  }
  
  .stat-icon {
    width: 60rpx;
    height: 60rpx;
    font-size: 36rpx;
  }
  
  .stat-label {
    font-size: 22rpx;
  }
  
  .stat-value {
    font-size: 36rpx;
  }
  
  .sidebar-menu {
    display: none;
  }
  
  /* 主内容区调整 */
  .main-layout {
    padding-left: 0;
    padding-top: 280px;
    padding-right: 16rpx;
  }
  
  .content-area {
    padding: 16rpx;
  }
  
  /* 顶部标题栏 */
  .top-header {
    padding: 0 16rpx;
  }
  
  .header-brand-icon {
    width: 36px;
    height: 36px;
  }
  
  .header-brand-title {
    font-size: 15px;
  }
  
  .header-brand-subtitle {
    display: none;
  }
  
  .profile-info {
    display: none;
  }
  
  /* 论文卡片 */
  .paper-card {
    padding: 20rpx;
    margin-bottom: 16rpx;
  }
  
  .paper-header {
    flex-direction: column;
    gap: 16rpx;
  }
  
  .paper-title-section {
    width: 100%;
  }
  
  .paper-title {
    font-size: 28rpx;
    line-height: 1.4;
  }
  
  .paper-meta {
    flex-wrap: wrap;
    gap: 12rpx;
  }
  
  .paper-header-actions {
    width: 100%;
    justify-content: flex-end;
  }
  
  /* 卡片内容 */
  .paper-content-wrapper {
    padding: 16rpx 0;
  }
  
  .paper-actions {
    flex-wrap: wrap;
    gap: 12rpx;
  }
  
  .action-btn {
    flex: 1;
    min-width: 120rpx;
    padding: 16rpx 20rpx;
    font-size: 24rpx;
  }
  
  /* 弹窗适配 */
  .annotation-modal-content {
    width: 95%;
    max-width: 95%;
    height: 85vh;
  }
  
  .update-modal-content,
  .review-modal-content {
    width: 90%;
    max-width: 90%;
  }
  
  .annotation-modal-header,
  .update-modal-header,
  .review-modal-header {
    height: 56px;
    padding: 0 16rpx;
  }
  
  .annotation-modal-title,
  .update-modal-title,
  .review-modal-title {
    font-size: 15px;
  }
  
  /* 空状态 */
  .empty-container {
    padding: 80rpx 40rpx;
  }
  
  .empty-icon {
    font-size: 64rpx;
  }
  
  .empty-text {
    font-size: 28rpx;
  }
  
  .empty-subtext {
    font-size: 24rpx;
  }
}

/* 超小屏幕 (<480px) */
@media screen and (max-width: 479px) {
  .sidebar {
    max-height: 180px;
    padding: 12rpx;
  }
  
  .main-layout {
    padding-top: 240px;
  }
  
  .stat-item {
    min-width: 120rpx;
    padding: 16rpx 12rpx;
  }
  
  .stat-icon {
    width: 48rpx;
    height: 48rpx;
    font-size: 28rpx;
  }
  
  .stat-label {
    font-size: 20rpx;
  }
  
  .stat-value {
    font-size: 32rpx;
  }
  
  .paper-card {
    padding: 16rpx;
  }
  
  .paper-title {
    font-size: 26rpx;
  }
  
  .action-btn {
    padding: 14rpx 16rpx;
    font-size: 22rpx;
  }
  
  .action-btn .btn-icon {
    font-size: 22rpx;
  }
}

/* 窗口缩放时的过渡动画 */
.sidebar,
.main-layout,
.paper-card,
.stat-item {
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

/* Profile Button Wrapper */
.profile-btn-wrapper {
  position: relative;
  display: flex;
  align-items: center;
  gap: var(--spacing-3);
  cursor: pointer;
  padding: var(--spacing-1) var(--spacing-3) var(--spacing-1) var(--spacing-1);
  border-radius: var(--radius-full);
  transition: background 0.2s;
}

.profile-btn-wrapper:hover {
  background: var(--surface-container-low);
}

/* User Info Card */
.user-info-card {
  position: absolute;
  top: calc(100% + 8px);
  right: 0;
  width: 380px;
  background: var(--surface-container-lowest);
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-ambient);
  z-index: 100;
  overflow: hidden;
  opacity: 0;
  transform: translateY(-10px) scale(0.95);
  transform-origin: top right;
  transition: opacity 0.3s ease, transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  pointer-events: none;
  visibility: hidden;
}

.user-info-card.show {
  opacity: 1;
  transform: translateY(0) scale(1);
  pointer-events: auto;
  visibility: visible;
}

.user-card-header {
  padding: var(--spacing-6);
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-container) 100%);
  display: flex;
  align-items: center;
  gap: var(--spacing-5);
}

.user-card-avatar {
  width: 56px;
  height: 56px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 22px;
  font-weight: 600;
  border: 2px solid rgba(255, 255, 255, 0.3);
}

.user-card-info {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-2);
}

.user-card-info .user-name {
  font-size: 17px;
  font-weight: 600;
  color: white;
}

.user-card-info .user-id {
  font-size: 13px;
  font-weight: 400;
  color: rgba(255, 255, 255, 0.8);
}

.user-role-badge {
  display: inline-flex;
  padding: 4px 12px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: var(--radius-full);
  font-size: 12px;
  font-weight: 500;
  color: white;
  width: fit-content;
}

.user-card-menu {
  padding: var(--spacing-3);
  display: flex;
  flex-direction: column;
  gap: var(--spacing-2);
}

.user-card-menu-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 16px;
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: background 0.2s;
  font-size: 15px;
  font-weight: 500;
  color: var(--on-surface);
}

.user-card-menu-item:hover {
  background: var(--surface-container-low);
}

.user-card-menu-item.logout {
  color: var(--error);
}

.user-card-menu-item.logout:hover {
  background: var(--error-container);
}

.user-card-menu-item .material-symbols-outlined {
  font-size: 20px;
  color: var(--on-surface-variant);
}

.user-card-menu-item.logout .material-symbols-outlined {
  color: var(--error);
}

/* ===== 弹窗样式 ===== */
.modal-backdrop {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  animation: modalBackdropIn 0.3s ease;
}

@keyframes modalBackdropIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.modal-content {
  background: var(--surface-container-lowest);
  border-radius: var(--radius-lg);
  width: 90%;
  max-width: 400px;
  max-height: 80vh;
  overflow: hidden;
  box-shadow: var(--shadow-ambient);
  animation: modalSlideIn 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  display: flex;
  flex-direction: column;
}

/* 修改密码弹窗特定样式 - 确保优先级 */
.password-modal-content,
.about-modal {
  width: 90%;
  max-width: 400px !important;
  max-height: 80vh;
}

.password-modal-content .modal-header,
.about-modal .modal-header {
  padding: var(--spacing-4) var(--spacing-5);
  background: var(--surface-container-low);
  border-radius: var(--radius-lg) var(--radius-lg) 0 0;
  min-height: auto;
}

.password-modal-content .modal-body {
  padding: var(--spacing-5);
}

.password-modal-content .modal-footer,
.about-modal .modal-footer {
  padding: var(--spacing-4) var(--spacing-5);
  gap: var(--spacing-3);
  background: var(--surface-container-low);
  border-radius: 0 0 var(--radius-lg) var(--radius-lg);
}

@keyframes modalSlideIn {
  from {
    opacity: 0;
    transform: translateY(-30px) scale(0.95);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--spacing-4) var(--spacing-5);
  background: var(--surface-container-low);
  border-radius: var(--radius-lg) var(--radius-lg) 0 0;
}

.modal-title {
  font-size: 1.125rem;
  font-weight: 600;
  font-family: var(--font-body);
  color: var(--on-surface);
}

.modal-close {
  font-size: 1.5rem;
  color: var(--on-surface-variant);
  cursor: pointer;
  padding: var(--spacing-1);
  transition: color 0.2s;
}

.modal-close:hover {
  color: var(--on-surface);
}

.modal-body {
  padding: var(--spacing-5);
  flex: 1;
  overflow-y: auto;
}

.modal-footer {
  display: flex;
  padding: var(--spacing-4) var(--spacing-5);
  gap: var(--spacing-3);
  background: var(--surface-container-low);
  border-radius: 0 0 var(--radius-lg) var(--radius-lg);
}

/* 表单样式 */
.form-item {
  margin-bottom: var(--spacing-4);
}

.form-label {
  display: block;
  font-size: 0.875rem;
  font-weight: 500;
  font-family: var(--font-body);
  color: var(--on-surface);
  margin-bottom: var(--spacing-2);
}

.form-input {
  width: 100%;
  height: 44px;
  padding: 0 var(--spacing-4);
  border: none;
  border-radius: var(--radius-md);
  font-size: 0.875rem;
  font-family: var(--font-body);
  color: var(--on-surface);
  background: var(--surface-container-low);
  box-sizing: border-box;
  transition: all 0.2s;
}

.form-input:focus {
  background: var(--surface-container-high);
  outline: none;
}

.form-tips {
  margin-top: var(--spacing-3);
  padding: var(--spacing-3);
  background: var(--amber-tint);
  border-radius: var(--radius-md);
}

.form-tips.error-tips {
  background: var(--error-container);
}

.tips-text {
  font-size: 0.75rem;
  font-weight: 400;
  font-family: var(--font-body);
  color: var(--on-amber);
}

.tips-text.error-text {
  color: var(--on-error-container);
}

/* 按钮样式 */
.btn {
  flex: 1;
  height: 44px;
  border-radius: var(--radius-md);
  font-size: 0.875rem;
  font-weight: 600;
  font-family: var(--font-body);
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.btn-cancel {
  background: var(--surface-container-high);
  color: var(--on-surface-variant);
}

.btn-cancel:hover {
  background: var(--surface-container-low);
}

.btn-confirm {
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-container) 100%);
  color: white;
  box-shadow: var(--shadow-primary);
}

.btn-confirm:hover {
  box-shadow: 0 6px 20px rgba(0, 91, 191, 0.35);
  transform: translateY(-1px);
}

/* 关于系统弹窗 */
.about-modal-body {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: var(--spacing-8);
  text-align: center;
}

.about-icon {
  width: 64px;
  height: 64px;
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-container) 100%);
  border-radius: var(--radius-lg);
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: var(--spacing-4);
}

.about-icon .material-symbols-outlined {
  font-size: 32px;
  color: white;
}

.about-title {
  font-size: 1.125rem;
  font-weight: 600;
  font-family: var(--font-body);
  color: var(--on-surface);
  margin-bottom: var(--spacing-2);
}

.about-version {
  font-size: 0.875rem;
  font-weight: 400;
  font-family: var(--font-body);
  color: var(--on-surface-variant);
  margin-bottom: var(--spacing-4);
}

.about-desc {
  font-size: 0.875rem;
  font-weight: 400;
  font-family: var(--font-body);
  color: var(--on-surface-variant);
  line-height: 1.6;
}

.agent-permission-modal {
  width: 90%;
  max-width: 420px !important;
  border: 1px solid rgba(26, 115, 232, 0.12);
  box-shadow: 0 20px 48px rgba(15, 23, 42, 0.16);
}

.agent-permission-header {
  background: linear-gradient(135deg, rgba(0, 91, 191, 0.12) 0%, rgba(26, 115, 232, 0.18) 100%);
  border-bottom: 1px solid rgba(26, 115, 232, 0.1);
}

.agent-permission-header .modal-title {
  color: var(--primary);
  font-weight: 700;
}

.agent-permission-header .modal-close {
  color: var(--primary);
}

.agent-permission-modal-body {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: var(--spacing-8) var(--spacing-6) var(--spacing-7);
  text-align: center;
  gap: var(--spacing-4);
  background: linear-gradient(180deg, rgba(26, 115, 232, 0.04) 0%, rgba(255, 255, 255, 0) 40%);
}

.agent-permission-icon {
  width: 72px;
  height: 72px;
  border-radius: 24px;
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-container) 100%);
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: var(--shadow-primary);
  border: 6px solid rgba(255, 255, 255, 0.88);
}

.agent-permission-icon .material-symbols-outlined {
  font-size: 34px;
  color: white;
}

.agent-permission-title {
  font-size: 1.125rem;
  font-weight: 700;
  font-family: var(--font-body);
  color: var(--on-surface);
}

.agent-permission-student {
  padding: 8px 16px;
  border-radius: var(--radius-full);
  background: rgba(0, 91, 191, 0.08);
  color: var(--primary);
  font-size: 0.875rem;
  font-weight: 600;
  font-family: var(--font-body);
}

.agent-permission-desc {
  font-size: 0.9rem;
  line-height: 1.7;
  color: var(--on-surface-variant);
  font-family: var(--font-body);
  max-width: 320px;
}

.agent-permission-tip {
  width: 100%;
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 14px 16px;
  border-radius: var(--radius-md);
  background: linear-gradient(135deg, rgba(254, 243, 199, 0.92) 0%, #fffaf0 100%);
  border: 1px solid rgba(245, 158, 11, 0.18);
  color: var(--on-amber);
  box-sizing: border-box;
  text-align: left;
}

.agent-permission-tip .material-symbols-outlined {
  font-size: 20px;
  line-height: 1.2;
  flex-shrink: 0;
}

.agent-permission-tip text:last-child {
  font-size: 0.875rem;
  line-height: 1.6;
  font-family: var(--font-body);
}

.agent-permission-footer {
  background: linear-gradient(180deg, rgba(26, 115, 232, 0.03) 0%, var(--surface-container-low) 100%);
  border-top: 1px solid rgba(26, 115, 232, 0.08);
}

.agent-use-modal {
  width: 90%;
  max-width: 420px !important;
  border: 1px solid rgba(8, 151, 156, 0.16);
  box-shadow: 0 20px 48px rgba(15, 23, 42, 0.18);
}

.agent-use-header {
  background: linear-gradient(135deg, rgba(8, 151, 156, 0.12) 0%, rgba(56, 178, 172, 0.2) 100%);
  border-bottom: 1px solid rgba(8, 151, 156, 0.14);
}

.agent-use-header .modal-title,
.agent-use-header .modal-close {
  color: #0f766e;
}

.agent-use-modal-body {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: var(--spacing-8) var(--spacing-6) var(--spacing-7);
  text-align: center;
  gap: var(--spacing-4);
  background: linear-gradient(180deg, rgba(56, 178, 172, 0.05) 0%, rgba(255, 255, 255, 0) 42%);
}

.agent-use-icon {
  width: 72px;
  height: 72px;
  border-radius: 24px;
  background: linear-gradient(135deg, #08979c 0%, #38b2ac 100%);
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 14px 28px rgba(8, 151, 156, 0.22);
  border: 6px solid rgba(255, 255, 255, 0.88);
}

.agent-use-icon .material-symbols-outlined {
  font-size: 34px;
  color: white;
}

.agent-use-title {
  font-size: 1.125rem;
  font-weight: 700;
  font-family: var(--font-body);
  color: var(--on-surface);
}

.agent-use-student,
.agent-use-paper {
  padding: 8px 16px;
  border-radius: var(--radius-full);
  background: rgba(8, 151, 156, 0.08);
  color: #0f766e;
  font-size: 0.875rem;
  font-weight: 600;
  font-family: var(--font-body);
}

.agent-use-paper {
  max-width: 100%;
  word-break: break-word;
}

.agent-use-desc {
  font-size: 0.9rem;
  line-height: 1.7;
  color: var(--on-surface-variant);
  font-family: var(--font-body);
  max-width: 320px;
}

.agent-use-tip {
  width: 100%;
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 14px 16px;
  border-radius: var(--radius-md);
  background: linear-gradient(135deg, rgba(230, 255, 251, 0.96) 0%, rgba(240, 253, 250, 1) 100%);
  border: 1px solid rgba(8, 151, 156, 0.16);
  color: #0f766e;
  box-sizing: border-box;
  text-align: left;
}

.agent-use-tip .material-symbols-outlined {
  font-size: 20px;
  line-height: 1.2;
  flex-shrink: 0;
}

.agent-use-tip text:last-child {
  font-size: 0.875rem;
  line-height: 1.6;
  font-family: var(--font-body);
}

.agent-use-footer {
  background: linear-gradient(180deg, rgba(8, 151, 156, 0.03) 0%, var(--surface-container-low) 100%);
  border-top: 1px solid rgba(8, 151, 156, 0.08);
}

/* 提交审查按鈕：青绿主题，带图标 */
.agent-use-submit-btn {
  display: flex !important;
  align-items: center;
  gap: 6px;
  background: linear-gradient(135deg, #08979c 0%, #38b2ac 100%) !important;
  box-shadow: 0 4px 14px rgba(8, 151, 156, 0.28) !important;
}

.agent-use-submit-btn .material-symbols-outlined {
  font-size: 17px;
}

/* 智能体弹窗 - 加载动画 */
@keyframes agent-spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.agent-spin {
  animation: agent-spin 1s linear infinite;
  display: inline-block;
}

.agent-use-icon-loading {
  background: linear-gradient(135deg, #6366f1 0%, #8b5cf6 100%) !important;
  box-shadow: 0 14px 28px rgba(99, 102, 241, 0.22) !important;
}

.agent-use-icon-error {
  background: linear-gradient(135deg, #ef4444 0%, #f97316 100%) !important;
  box-shadow: 0 14px 28px rgba(239, 68, 68, 0.22) !important;
}

.agent-use-title-error {
  color: #dc2626 !important;
}

.agent-use-error-msg {
  font-size: 0.875rem;
  color: #dc2626;
  background: rgba(239, 68, 68, 0.08);
  padding: 10px 16px;
  border-radius: var(--radius-lg, 12px);
  width: 100%;
  text-align: center;
  font-family: var(--font-body);
  box-sizing: border-box;
}

.agent-use-success-tip {
  background: rgba(16, 185, 129, 0.08) !important;
  border: 1px solid rgba(16, 185, 129, 0.2) !important;
  color: #065f46 !important;
}

.agent-use-success-tip .material-symbols-outlined {
  color: #10b981 !important;
}

.agent-use-warn-tip {
  background: rgba(245, 158, 11, 0.08) !important;
  border: 1px solid rgba(245, 158, 11, 0.2) !important;
  color: #92400e !important;
}

.agent-use-warn-tip .material-symbols-outlined {
  color: #f59e0b !important;
}

/* 后端 MySQL异常但实际成功的提示（靖蓝色） */
.agent-use-verified-tip {
  background: rgba(79, 70, 229, 0.07) !important;
  border: 1px solid rgba(79, 70, 229, 0.18) !important;
  color: #3730a3 !important;
}

.agent-use-verified-tip .material-symbols-outlined {
  color: #6366f1 !important;
}

.agent-request-modal {
  width: 90%;
  max-width: 430px !important;
  border: 1px solid rgba(217, 119, 6, 0.14);
  box-shadow: 0 20px 48px rgba(15, 23, 42, 0.16);
}

.agent-request-header {
  background: linear-gradient(135deg, rgba(245, 158, 11, 0.12) 0%, rgba(251, 191, 36, 0.18) 100%);
  border-bottom: 1px solid rgba(217, 119, 6, 0.1);
}

.agent-request-header .modal-title,
.agent-request-header .modal-close {
  color: #b45309;
}

.agent-request-modal-body {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: var(--spacing-8) var(--spacing-6) var(--spacing-7);
  text-align: center;
  gap: var(--spacing-4);
  background: linear-gradient(180deg, rgba(251, 191, 36, 0.05) 0%, rgba(255, 255, 255, 0) 40%);
}

.agent-request-modal-body.success-state {
  gap: var(--spacing-3);
}

.agent-request-icon {
  width: 72px;
  height: 72px;
  border-radius: 24px;
  background: linear-gradient(135deg, #d97706 0%, #f59e0b 100%);
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 14px 28px rgba(217, 119, 6, 0.22);
  border: 6px solid rgba(255, 255, 255, 0.88);
}

.agent-request-icon.success {
  background: linear-gradient(135deg, #08979c 0%, #38b2ac 100%);
  box-shadow: 0 14px 28px rgba(8, 151, 156, 0.22);
}

.agent-request-icon .material-symbols-outlined {
  font-size: 34px;
  color: #fff;
}

.agent-request-title {
  font-size: 1.125rem;
  font-weight: 700;
  font-family: var(--font-body);
  color: var(--on-surface);
}

.agent-request-desc {
  max-width: 320px;
  font-size: 0.9rem;
  line-height: 1.7;
  color: var(--on-surface-variant);
  font-family: var(--font-body);
}

.agent-request-student,
.agent-request-badge {
  padding: 8px 16px;
  border-radius: var(--radius-full);
  background: rgba(217, 119, 6, 0.08);
  color: #b45309;
  font-size: 0.875rem;
  font-weight: 600;
  font-family: var(--font-body);
  max-width: 100%;
  word-break: break-word;
}

.agent-request-form {
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: var(--spacing-2);
  text-align: left;
}

.agent-request-label {
  font-size: 0.875rem;
  font-weight: 600;
  font-family: var(--font-body);
  color: var(--on-surface);
}

.agent-request-input {
  width: 100%;
  height: 46px;
  padding: 0 var(--spacing-4);
  border: none;
  border-radius: var(--radius-md);
  font-size: 0.9rem;
  font-family: var(--font-body);
  color: var(--on-surface);
  background: var(--surface-container-low);
  box-sizing: border-box;
  transition: all 0.2s;
}

.agent-request-input:focus {
  background: var(--surface-container-high);
  outline: none;
}

.agent-request-placeholder {
  color: var(--on-surface-variant);
}

.agent-request-error {
  font-size: 0.75rem;
  color: #b3261e;
  font-family: var(--font-body);
  line-height: 1.5;
}

.agent-request-tip {
  width: 100%;
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 14px 16px;
  border-radius: var(--radius-md);
  background: linear-gradient(135deg, rgba(254, 243, 199, 0.94) 0%, #fffaf0 100%);
  border: 1px solid rgba(245, 158, 11, 0.16);
  color: var(--on-amber);
  box-sizing: border-box;
  text-align: left;
}

.agent-request-tip.success-tip {
  background: linear-gradient(135deg, rgba(230, 255, 251, 0.96) 0%, rgba(240, 253, 250, 1) 100%);
  border-color: rgba(8, 151, 156, 0.16);
  color: #0f766e;
}

.agent-request-tip .material-symbols-outlined {
  font-size: 20px;
  line-height: 1.2;
  flex-shrink: 0;
}

.agent-request-tip text:last-child {
  font-size: 0.875rem;
  line-height: 1.6;
  font-family: var(--font-body);
}

.agent-request-footer {
  background: linear-gradient(180deg, rgba(245, 158, 11, 0.03) 0%, var(--surface-container-low) 100%);
  border-top: 1px solid rgba(245, 158, 11, 0.08);
}

.agent-request-footer .btn.disabled,
.agent-request-footer .btn.disabled:hover {
  opacity: 0.7;
  cursor: not-allowed;
  transform: none;
  box-shadow: var(--shadow-primary);
}

/* ===== 工作台同款用户菜单样式（放在文件尾部以覆盖页面通用规则） ===== */
.student-profile-btn {
  position: relative;
  display: flex;
  align-items: center;
  gap: var(--spacing-3);
  cursor: pointer;
  padding: var(--spacing-1) var(--spacing-3) var(--spacing-1) var(--spacing-1);
  border-radius: var(--radius-full);
  transition: background 0.2s;
}

.student-profile-btn:hover {
  background: var(--surface-container-low);
}

.student-profile-btn .profile-avatar {
  width: 36px;
  height: 36px;
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-container) 100%);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 14px;
  font-weight: 600;
  font-family: var(--font-display);
}

.student-profile-btn .profile-name {
  font-size: 16px;
  font-weight: 600;
  font-family: var(--font-body);
  color: var(--on-surface);
}

.student-profile-btn .profile-role {
  font-size: 12px;
  font-weight: 400;
  font-family: var(--font-body);
  color: var(--on-surface-variant);
}

.student-user-card {
  position: absolute;
  top: calc(100% + 8px);
  right: 0;
  width: 380px;
  background: var(--surface-container-lowest);
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-ambient);
  z-index: 100;
  overflow: hidden;
  opacity: 0;
  transform: translateY(-10px) scale(0.95);
  transform-origin: top right;
  transition: opacity 0.3s ease, transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  pointer-events: none;
  visibility: hidden;
}

.student-user-card.show {
  opacity: 1;
  transform: translateY(0) scale(1);
  pointer-events: auto;
  visibility: visible;
}

.student-user-card .user-card-header {
  padding: var(--spacing-6);
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-container) 100%);
  display: flex;
  align-items: center;
  gap: var(--spacing-5);
}

.student-user-card .user-card-avatar {
  width: 56px;
  height: 56px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 22px;
  font-weight: 600;
  font-family: var(--font-display);
  border: 2px solid rgba(255, 255, 255, 0.3);
}

.student-user-card .user-card-info {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-2);
}

.student-user-card .user-card-info .user-name {
  font-size: 17px;
  font-weight: 600;
  font-family: var(--font-body);
  color: white;
}

.student-user-card .user-card-info .user-id {
  font-size: 13px;
  font-weight: 400;
  font-family: var(--font-body);
  color: rgba(255, 255, 255, 0.8);
}

.student-user-card .user-role-badge {
  display: inline-flex;
  padding: 4px 12px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: var(--radius-full);
  font-size: 12px;
  font-weight: 500;
  font-family: var(--font-body);
  color: white;
  width: fit-content;
}

.student-user-card .user-card-menu {
  padding: var(--spacing-3);
  display: flex;
  flex-direction: column;
  gap: var(--spacing-2);
}

.student-user-card .user-card-menu-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 16px;
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: background 0.2s;
  font-size: 15px;
  font-weight: 500;
  font-family: var(--font-body);
  color: var(--on-surface);
}

.student-user-card .user-card-menu-item:hover {
  background: var(--surface-container-low);
}

.student-user-card .user-card-menu-item.logout {
  color: var(--error);
}

.student-user-card .user-card-menu-item.logout:hover {
  background: var(--error-container);
}

.student-user-card .user-card-menu-item .material-symbols-outlined {
  font-size: 20px;
  color: var(--on-surface-variant);
}

.student-user-card .user-card-menu-item.logout .material-symbols-outlined {
  color: var(--error);
}

.student-user-panel-modal {
  background: var(--surface-container-lowest);
  border-radius: var(--radius-lg);
  width: 90%;
  max-width: 400px;
  max-height: 80vh;
  overflow: hidden;
  box-shadow: var(--shadow-ambient);
  animation: modalSlideIn 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  display: flex;
  flex-direction: column;
}

.student-user-panel-modal .user-panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--spacing-4) var(--spacing-5);
  background: var(--surface-container-low);
  border-radius: var(--radius-lg) var(--radius-lg) 0 0;
  min-height: auto;
}

.student-user-panel-modal .user-panel-header.logout-header {
  background: linear-gradient(135deg, #fc8181 0%, #f56565 100%);
}

.student-user-panel-modal .user-panel-header.logout-header .user-panel-title,
.student-user-panel-modal .user-panel-header.logout-header .user-panel-close {
  color: white;
}

.student-user-panel-modal .user-panel-title {
  font-size: 1.125rem;
  font-weight: 600;
  font-family: var(--font-body);
  color: var(--on-surface);
}

.student-user-panel-modal .user-panel-close {
  font-size: 1.5rem;
  color: var(--on-surface-variant);
  cursor: pointer;
  padding: var(--spacing-1);
  transition: color 0.2s;
}

.student-user-panel-modal .user-panel-close:hover {
  color: var(--on-surface);
}

.student-user-panel-modal .user-panel-body {
  padding: var(--spacing-5);
  flex: 1;
  overflow-y: auto;
}

.student-user-panel-modal .user-panel-footer {
  display: flex;
  padding: var(--spacing-4) var(--spacing-5);
  gap: var(--spacing-3);
  background: var(--surface-container-low);
  border-radius: 0 0 var(--radius-lg) var(--radius-lg);
}

.student-user-panel-modal .user-panel-form-item {
  margin-bottom: var(--spacing-4);
}

.student-user-panel-modal .user-panel-form-label {
  display: block;
  font-size: 0.875rem;
  font-weight: 500;
  font-family: var(--font-body);
  color: var(--on-surface);
  margin-bottom: var(--spacing-2);
}

.student-user-panel-modal .user-panel-form-input {
  width: 100%;
  height: 44px;
  padding: 0 var(--spacing-4);
  border: none;
  border-radius: var(--radius-md);
  font-size: 0.875rem;
  font-family: var(--font-body);
  color: var(--on-surface);
  background: var(--surface-container-low);
  box-sizing: border-box;
  transition: all 0.2s;
}

.student-user-panel-modal .user-panel-form-input:focus {
  background: var(--surface-container-high);
  outline: none;
}

.student-user-panel-modal .user-panel-form-tips {
  margin-top: var(--spacing-3);
  padding: var(--spacing-3);
  background: var(--amber-tint);
  border-radius: var(--radius-md);
}

.student-user-panel-modal .user-panel-form-tips.error-tips {
  background: var(--error-container);
}

.student-user-panel-modal .user-panel-tips-text {
  font-size: 0.75rem;
  font-weight: 400;
  font-family: var(--font-body);
  color: var(--on-amber);
}

.student-user-panel-modal .user-panel-tips-text.error-text {
  color: var(--on-error-container);
}

.student-user-panel-modal .user-panel-btn {
  flex: 1;
  height: 44px;
  border-radius: var(--radius-md);
  font-size: 0.875rem;
  font-weight: 600;
  font-family: var(--font-body);
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.student-user-panel-modal .user-panel-btn-cancel {
  background: var(--surface-container-high);
  color: var(--on-surface-variant);
}

.student-user-panel-modal .user-panel-btn-cancel:hover {
  background: var(--surface-container-low);
}

.student-user-panel-modal .user-panel-btn-confirm {
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-container) 100%);
  color: white;
  box-shadow: var(--shadow-primary);
}

.student-user-panel-modal .user-panel-btn-confirm:hover {
  box-shadow: 0 6px 20px rgba(0, 91, 191, 0.35);
  transform: translateY(-1px);
}

.student-user-panel-modal .user-panel-btn-logout {
  background: linear-gradient(135deg, #fc8181 0%, #f56565 100%);
  color: white;
}

.student-user-panel-modal .user-panel-btn-logout:hover {
  box-shadow: 0 4px 12px rgba(245, 101, 101, 0.3);
}

.student-user-panel-modal .user-panel-about-body {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: var(--spacing-8);
  text-align: center;
}

.student-user-panel-modal .user-panel-about-icon {
  width: 64px;
  height: 64px;
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-container) 100%);
  border-radius: var(--radius-lg);
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: var(--spacing-4);
}

.student-user-panel-modal .user-panel-about-icon .material-symbols-outlined {
  font-size: 32px;
  color: white;
}

.student-user-panel-modal .user-panel-about-title {
  font-size: 1.125rem;
  font-weight: 600;
  font-family: var(--font-body);
  color: var(--on-surface);
  margin-bottom: var(--spacing-2);
}

.student-user-panel-modal .user-panel-about-version {
  font-size: 0.875rem;
  font-weight: 400;
  font-family: var(--font-body);
  color: var(--on-surface-variant);
  margin-bottom: var(--spacing-4);
}

.student-user-panel-modal .user-panel-about-desc {
  font-size: 0.875rem;
  font-weight: 400;
  font-family: var(--font-body);
  color: var(--on-surface-variant);
  line-height: 1.6;
}

/* ===== 智能体报告弹窗 ===== */
.agent-report-backdrop {
  z-index: 999;
}

.agent-report-modal {
  width: 92%;
  max-width: 640px !important;
  max-height: 88vh;
  display: flex;
  flex-direction: column;
  border: 1px solid rgba(79, 70, 229, 0.14);
  box-shadow: 0 24px 56px rgba(15, 23, 42, 0.18);
  animation: modalSlideIn 0.38s cubic-bezier(0.4, 0, 0.2, 1);
  overflow: hidden;
}

/* 弹窗头部 */
.agent-report-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 18px 24px;
  background: linear-gradient(135deg, rgba(79, 70, 229, 0.1) 0%, rgba(99, 102, 241, 0.16) 100%);
  border-bottom: 1px solid rgba(79, 70, 229, 0.1);
  flex-shrink: 0;
}

.agent-report-header-left {
  display: flex;
  align-items: center;
  gap: 14px;
}

.agent-report-header-icon {
  width: 44px;
  height: 44px;
  border-radius: 14px;
  background: linear-gradient(135deg, #4f46e5 0%, #6366f1 100%);
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 6px 16px rgba(79, 70, 229, 0.28);
  flex-shrink: 0;
}

.agent-report-header-icon .material-symbols-outlined {
  font-size: 22px;
  color: #fff;
}

.agent-report-header-text {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.agent-report-header-title {
  font-size: 1.0625rem;
  font-weight: 700;
  font-family: var(--font-body);
  color: #312e81;
}

.agent-report-header-sub {
  font-size: 0.78rem;
  font-family: var(--font-body);
  color: #6366f1;
}

.agent-report-header .modal-close {
  color: #4f46e5;
  font-size: 1.375rem;
  cursor: pointer;
  transition: color 0.2s;
  flex-shrink: 0;
}

.agent-report-header .modal-close:hover {
  color: #312e81;
}

/* 弹窗主体 */
.agent-report-body {
  flex: 1;
  overflow-y: auto;
  padding: 20px 24px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  background: linear-gradient(180deg, rgba(99, 102, 241, 0.03) 0%, var(--surface-container-lowest, #fff) 100%);
}

/* 全局加载状态 */
.agent-report-loading {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 48px 0;
  gap: 14px;
  color: #6366f1;
}

.agent-report-loading .material-symbols-outlined {
  font-size: 40px;
  color: #6366f1;
}

.agent-report-loading-text {
  font-size: 0.9rem;
  font-family: var(--font-body);
  color: var(--on-surface-variant);
}

/* 空状态 */
.agent-report-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 40px 20px;
  gap: 12px;
  text-align: center;
}

.agent-report-empty-icon {
  width: 64px;
  height: 64px;
  border-radius: 20px;
  background: linear-gradient(135deg, rgba(99, 102, 241, 0.1) 0%, rgba(139, 92, 246, 0.14) 100%);
  display: flex;
  align-items: center;
  justify-content: center;
}

.agent-report-empty-icon .material-symbols-outlined {
  font-size: 32px;
  color: #6366f1;
}

.agent-report-empty-title {
  font-size: 1rem;
  font-weight: 600;
  font-family: var(--font-body);
  color: var(--on-surface);
}

.agent-report-empty-desc {
  font-size: 0.875rem;
  font-family: var(--font-body);
  color: #b91c1c;
  line-height: 1.5;
}

.agent-report-empty-tip {
  font-size: 0.8rem;
  font-family: var(--font-body);
  color: var(--on-surface-variant);
}

/* 任务元信息卡片 */
.agent-report-task-card {
  background: var(--surface-container-low, #f8fafc);
  border-radius: 12px;
  border: 1px solid rgba(79, 70, 229, 0.1);
  padding: 14px 18px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.agent-report-task-row {
  display: flex;
  align-items: center;
  gap: 10px;
}

.agent-report-task-label {
  font-size: 0.8rem;
  font-family: var(--font-body);
  color: var(--on-surface-variant);
  font-weight: 500;
  min-width: 56px;
}

.agent-report-task-value {
  font-size: 0.875rem;
  font-family: var(--font-body);
  color: var(--on-surface);
}

.agent-report-mono {
  font-family: 'Menlo', 'Consolas', 'Fira Code', monospace;
  font-size: 0.82rem;
  color: #4f46e5;
  background: rgba(79, 70, 229, 0.06);
  padding: 2px 8px;
  border-radius: 6px;
  word-break: break-all;
}

/* 状态 badge */
.agent-report-status-badge {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 12px;
  border-radius: 999px;
  font-size: 0.8rem;
  font-weight: 600;
  font-family: var(--font-body);
  background: rgba(99, 102, 241, 0.1);
  color: #4338ca;
}

.agent-report-status-badge .material-symbols-outlined {
  font-size: 16px;
}

.agent-report-status-done,
.agent-report-status-completed,
.agent-report-status-success {
  background: rgba(16, 185, 129, 0.1) !important;
  color: #065f46 !important;
}

.agent-report-status-failed,
.agent-report-status-error {
  background: rgba(239, 68, 68, 0.1) !important;
  color: #991b1b !important;
}

.agent-report-status-pending,
.agent-report-status-processing,
.agent-report-status-running {
  background: rgba(245, 158, 11, 0.1) !important;
  color: #92400e !important;
}

/* 进度条 */
.agent-report-progress-wrap {
  display: flex;
  align-items: center;
  gap: 8px;
  flex: 1;
}

.agent-report-progress-bar {
  flex: 1;
  height: 6px;
  background: rgba(79, 70, 229, 0.12);
  border-radius: 999px;
  overflow: hidden;
}

.agent-report-progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #4f46e5, #818cf8);
  border-radius: 999px;
  transition: width 0.4s ease;
}

.agent-report-progress-pct {
  font-size: 0.8rem;
  font-family: var(--font-body);
  color: #4f46e5;
  font-weight: 600;
  min-width: 34px;
}

/* 错误提示条 */
.agent-report-error-tip {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  padding: 12px 16px;
  border-radius: 10px;
  background: rgba(239, 68, 68, 0.07);
  border: 1px solid rgba(239, 68, 68, 0.15);
  color: #b91c1c;
  font-size: 0.875rem;
  font-family: var(--font-body);
  line-height: 1.5;
}

.agent-report-error-tip .material-symbols-outlined {
  font-size: 18px;
  flex-shrink: 0;
  color: #ef4444;
}

/* 报告内容区 */
.agent-report-content {
  display: flex;
  flex-direction: column;
  gap: 18px;
}

/* 概览统计行 */
.agent-report-summary-row {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
}

.agent-report-stat-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 6px 14px;
  border-radius: 999px;
  background: rgba(79, 70, 229, 0.09);
  color: #4338ca;
  font-size: 0.82rem;
  font-weight: 600;
  font-family: var(--font-body);
}

.agent-report-stat-chip .material-symbols-outlined {
  font-size: 16px;
}

.agent-report-stat-chip.chip-teal {
  background: rgba(13, 148, 136, 0.09);
  color: #0f766e;
}

.agent-report-stat-chip.chip-purple {
  background: rgba(139, 92, 246, 0.09);
  color: #7c3aed;
}

/* 分区标题 */
.agent-report-section {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.agent-report-section-title {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.9rem;
  font-weight: 700;
  font-family: var(--font-body);
  color: var(--on-surface);
  padding-bottom: 8px;
  border-bottom: 1.5px solid rgba(79, 70, 229, 0.1);
}

.agent-report-section-title .material-symbols-outlined {
  font-size: 18px;
  color: #6366f1;
}

.agent-report-section-count {
  margin-left: auto;
  font-size: 0.78rem;
  font-weight: 500;
  color: var(--on-surface-variant);
  background: var(--surface-container-low, #f8fafc);
  padding: 2px 10px;
  border-radius: 999px;
}

/* 参考文献列表 */
.agent-report-ref-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.agent-report-ref-item {
  padding: 10px 14px;
  border-radius: 10px;
  background: var(--surface-container-low, #f8fafc);
  border: 1px solid transparent;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.agent-report-ref-item.ref-issue {
  background: rgba(239, 68, 68, 0.05);
  border-color: rgba(239, 68, 68, 0.15);
}

.agent-report-ref-top {
  display: flex;
  align-items: baseline;
  gap: 8px;
}

.agent-report-ref-num {
  font-size: 0.78rem;
  font-weight: 700;
  font-family: monospace;
  color: #6366f1;
  flex-shrink: 0;
}

.agent-report-ref-text {
  font-size: 0.85rem;
  font-family: var(--font-body);
  color: var(--on-surface);
  line-height: 1.5;
}

.agent-report-ref-remark {
  font-size: 0.78rem;
  font-family: var(--font-body);
  color: #b91c1c;
  line-height: 1.4;
  padding-left: 4px;
}

/* 筛选标签栏 */
.agent-report-filter-bar {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  padding: 4px 0;
}

.agent-report-filter-tag {
  padding: 5px 14px;
  border-radius: 999px;
  font-size: 0.8rem;
  font-weight: 500;
  font-family: var(--font-body);
  color: var(--on-surface-variant);
  background: var(--surface-container-low, #f8fafc);
  border: 1px solid transparent;
  cursor: pointer;
  transition: all 0.18s;
}

.agent-report-filter-tag.active {
  background: linear-gradient(135deg, #4f46e5 0%, #6366f1 100%);
  color: #fff;
  border-color: transparent;
  box-shadow: 0 2px 8px rgba(79, 70, 229, 0.22);
}

.agent-report-filter-tag:hover:not(.active) {
  background: rgba(79, 70, 229, 0.07);
  color: #4f46e5;
}

/* 分段列表 */
.agent-report-chunk-item {
  border: 1px solid rgba(79, 70, 229, 0.1);
  border-radius: 12px;
  overflow: hidden;
  background: var(--surface-container-lowest, #fff);
}

.agent-report-chunk-head {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 11px 16px;
  background: linear-gradient(135deg, rgba(79, 70, 229, 0.06) 0%, rgba(99, 102, 241, 0.1) 100%);
  font-size: 0.875rem;
  font-weight: 600;
  font-family: var(--font-body);
  color: #4338ca;
}

.agent-report-chunk-head .material-symbols-outlined {
  font-size: 17px;
  color: #6366f1;
}

.agent-report-chunk-count {
  margin-left: auto;
  font-size: 0.78rem;
  font-weight: 500;
  color: var(--on-surface-variant);
  background: rgba(255, 255, 255, 0.7);
  padding: 2px 10px;
  border-radius: 999px;
}

/* 单条问题 */
.agent-report-issue {
  padding: 12px 16px;
  border-top: 1px solid rgba(79, 70, 229, 0.07);
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.agent-report-issue-type-badge {
  display: inline-block;
  padding: 2px 10px;
  border-radius: 6px;
  font-size: 0.72rem;
  font-weight: 700;
  font-family: var(--font-body);
  background: rgba(79, 70, 229, 0.08);
  color: #4f46e5;
  align-self: flex-start;
}

.agent-report-issue-msg {
  font-size: 0.875rem;
  font-family: var(--font-body);
  color: var(--on-surface);
  line-height: 1.6;
}

.agent-report-issue-sug {
  font-size: 0.82rem;
  font-family: var(--font-body);
  color: #0f766e;
  background: rgba(13, 148, 136, 0.07);
  padding: 5px 10px;
  border-radius: 7px;
  line-height: 1.5;
}

.agent-report-chunk-empty {
  padding: 12px 16px;
  border-top: 1px solid rgba(79, 70, 229, 0.07);
  font-size: 0.82rem;
  font-family: var(--font-body);
  color: var(--on-surface-variant);
  text-align: center;
}

/* 无问题 */
.agent-report-no-issues {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  padding: 32px 0;
  color: #059669;
  text-align: center;
}

.agent-report-no-issues .material-symbols-outlined {
  font-size: 40px;
  color: #10b981;
}

.agent-report-no-issues text:last-child {
  font-size: 0.9rem;
  font-family: var(--font-body);
  color: #065f46;
}

/* 等待提示 */
.agent-report-pending-hint {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  padding: 36px 0;
  color: var(--on-surface-variant);
  text-align: center;
}

.agent-report-pending-hint .material-symbols-outlined {
  font-size: 38px;
  color: #f59e0b;
}

.agent-report-pending-hint text:last-child {
  font-size: 0.875rem;
  font-family: var(--font-body);
  line-height: 1.6;
}

/* 弹窗底部 */
.agent-report-footer {
  background: linear-gradient(180deg, rgba(79, 70, 229, 0.03) 0%, var(--surface-container-low) 100%);
  border-top: 1px solid rgba(79, 70, 229, 0.08);
  flex-shrink: 0;
}

.agent-report-query-btn,
.agent-report-refresh-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  background: linear-gradient(135deg, #4f46e5 0%, #6366f1 100%) !important;
  box-shadow: 0 4px 14px rgba(79, 70, 229, 0.28) !important;
}

.agent-report-query-btn.disabled,
.agent-report-refresh-btn.disabled {
  opacity: 0.65;
  cursor: not-allowed;
  transform: none !important;
}

.agent-report-query-btn .material-symbols-outlined,
.agent-report-refresh-btn .material-symbols-outlined {
  font-size: 17px;
}

</style>
