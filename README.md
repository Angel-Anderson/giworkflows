#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import requests
import random
import time
import json
import logging
from datetime import datetime
import os
from concurrent.futures import ThreadPoolExecutor, as_completed
import threading

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# 多账号配置（请修改为您的实际账号）
ACCOUNTS = [
    {"username": "jiliuquan001@163.com", "password": "liu12345"},
    {"username": "您的手机号2", "password": "您的密码2"},
]

# 步数范围配置（可根据时间段智能生成步数）
STEP_RANGES = {
    8: {"min": 6000, "max": 10000},
    12: {"min": 8000, "max": 14000},
    16: {"min": 10000, "max": 18000},
    20: {"min": 12000, "max": 22000},
    22: {"min": 15000, "max": 24000}
}

# 默认步数（当不在指定时间段时使用）
DEFAULT_STEPS = 18000

class StepSubmitter:
    def __init__(self):
        # 使用线程本地存储确保每个线程有独立的session[6](@ref)
        self._local = threading.local()
        self.base_url = 'https://wzz.wangzouzou.com/motion/api/motion/Xiaomi'
        
    def get_session(self):
        """为每个线程创建独立的session[6](@ref)"""
        if not hasattr(self._local, "session"):
            session = requests.Session()
            # 设置浏览器般的请求头
            session.headers.update({
                'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.7339.128 Safari/537.36',
                'Accept': 'application/json, text/javascript, */*; q=0.01',
                'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8',
                'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8',
                'Origin': 'https://m.cqzz.top',
                'Referer': 'https://m.cqzz.top/',
                'X-Requested-With': 'XMLHttpRequest'
            })
            self._local.session = session
        return self._local.session
        
    def get_current_steps(self):
        """根据当前时间获取对应的步数范围[1](@ref)"""
        current_hour = datetime.now().hour
        logger.info(f"当前时间: {datetime.now()}, 小时: {current_hour}")
        
        # 找到最接近的配置时间段
        closest_hour = None
        min_diff = float('inf')
        
        for hour in STEP_RANGES.keys():
            diff = abs(current_hour - hour)
            if diff < min_diff:
                min_diff = diff
                closest_hour = hour
        
        # 如果找到接近的配置且在合理范围内（2小时内），使用该配置
        if min_diff <= 2 and closest_hour in STEP_RANGES:
            step_config = STEP_RANGES[closest_hour]
            steps = random.randint(step_config['min'], step_config['max'])
            logger.info(f"使用 {closest_hour} 点配置，生成步数: {steps}")
        else:
            steps = DEFAULT_STEPS
            logger.info(f"使用默认步数: {steps}")
        
        return steps
     
    def validate_credentials(self, username, password):
        """验证账号密码格式（模拟前端验证）"""
        import re
        
        # 手机号正则
        phone_pattern = r'^1[3-9]\d{9}$'
        # 邮箱正则（简化版）
        email_pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        
        if not username or not password:
            return False, "账号或密码不能为空"
        
        if ' ' in password:
            return False, "密码不能包含空格"
        
        if re.match(phone_pattern, username) or re.match(email_pattern, username):
            return True, "验证通过"
        else:
            return False, "账号格式错误（需为手机号或邮箱）"
    
    def submit_single_account(self, account):
        """处理单个账号的步数提交任务[2](@ref)"""
        username = account['username']
        password = account['password']
        
        try:
            # 获取当前应提交的步数
            steps = self.get_current_steps()
            
            # 先验证凭证格式
            is_valid, message = self.validate_credentials(username, password)
            if not is_valid:
                return False, f"账号 {username} 验证失败: {message}"
            
            # 准备请求数据
            data = {
                'phone': username,
                'pwd': password,
                'num': steps
            }
            
            logger.info(f"准备提交 - 账号: {username}, 步数: {steps}")
            
            # 使用线程独立的session发送请求[6](@ref)
            session = self.get_session()
            response = session.post(
                self.base_url,
                data=data,
                timeout=30
            )
            
            # 解析响应
            if response.status_code == 200:
                result = response.json()
                if result.get('code') == 200:
                    return True, f"账号 {username} 提交成功! 步数: {steps}"
                else:
                    error_msg = result.get('data', '未知错误')
                    # 处理频繁提交的情况
                    if '频繁' in error_msg:
                        return False, f"账号 {username} 提交过于频繁，请稍后再试"
                    else:
                        return False, f"账号 {username} 提交失败: {error_msg}"
            else:
                return False, f"账号 {username} HTTP错误: {response.status_code}"
                
        except requests.exceptions.RequestException as e:
            return False, f"账号 {username} 网络请求错误: {str(e)}"
        except json.JSONDecodeError:
            return False, f"账号 {username} 响应解析错误"
        except Exception as e:
            return False, f"账号 {username} 未知错误: {str(e)}"
    
    def run_sequential(self):
        """顺序执行版本（原有逻辑）"""
        logger.info("开始顺序执行步数提交任务")
        logger.info(f"共有 {len(ACCOUNTS)} 个账号需要处理")
        
        success_count = 0
        fail_count = 0
        
        for i, account in enumerate(ACCOUNTS, 1):
            logger.info(f"处理第 {i}/{len(ACCOUNTS)} 个账号: {account['username']}")
            
            success, message = self.submit_single_account(account)
            
            if success:
                success_count += 1
                logger.info(f"✅ {message}")
            else:
                fail_count += 1
                logger.error(f"❌ {message}")
            
            # 账号间间隔（最后一个账号不需要等待）
            if i < len(ACCOUNTS):
                logger.info("等待3秒后处理下一个账号...")
                time.sleep(3)
        
        return success_count, fail_count
    
    def run_parallel(self, max_workers=3):
        """使用线程池并发执行多账号任务[2,6](@ref)"""
        logger.info(f"开始并发执行步数提交任务，最大线程数: {max_workers}")
        logger.info(f"共有 {len(ACCOUNTS)} 个账号需要处理")
        
        success_count = 0
        fail_count = 0
        
        # 使用ThreadPoolExecutor创建线程池[6](@ref)
        with ThreadPoolExecutor(max_workers=max_workers) as executor:
            # 提交所有任务到线程池[7](@ref)
            future_to_account = {
                executor.submit(self.submit_single_account, account): account 
                for account in ACCOUNTS
            }
            
            # 等待任务完成并收集结果[2](@ref)
            for future in as_completed(future_to_account):
                account = future_to_account[future]
                try:
                    success, message = future.result()
                    if success:
                        success_count += 1
                        logger.info(f"✅ {message}")
                    else:
                        fail_count += 1
                        logger.error(f"❌ {message}")
                except Exception as exc:
                    fail_count += 1
                    logger.error(f"❌ 账号 {account['username']} 处理异常: {str(exc)}")
        
        return success_count, fail_count
    
    def run(self, parallel=True, max_workers=3):
        """主执行函数"""
        if parallel and len(ACCOUNTS) > 1:
            return self.run_parallel(max_workers)
        else:
            return self.run_sequential()

def main():
    """主函数"""
    try:
        submitter = StepSubmitter()
        
        # 选择执行模式：并行（True）或顺序（False）
        # max_workers 控制并发线程数，建议3-5个[6](@ref)
        use_parallel = True
        max_workers = 3
        
        start_time = time.time()
        
        if use_parallel:
            success_count, fail_count = submitter.run(parallel=True, max_workers=max_workers)
        else:
            success_count, fail_count = submitter.run(parallel=False)
        
        end_time = time.time()
        execution_time = end_time - start_time
        
        # 汇总结果
        logger.info(f"任务完成! 成功: {success_count}, 失败: {fail_count}, 耗时: {execution_time:.2f}秒")
        
        # 返回结果（适用于青龙面板等环境）
        if fail_count == 0:
            print("所有账号提交成功!")
            exit(0)
        else:
            print(f"部分账号提交失败，成功: {success_count}, 失败: {fail_count}")
            exit(1)
            
    except Exception as e:
        logger.error(f"脚本执行异常: {str(e)}")
        exit(1)

if __name__ == "__main__":
    main()
