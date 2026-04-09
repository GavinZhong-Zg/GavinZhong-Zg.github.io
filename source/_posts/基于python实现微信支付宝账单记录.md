---

title: 基于python实现微信支付宝账单记录
date: 2026-4-9
tags: 
  - python编程
  - 账本
categories: 
  - 编程学习
keywords: 
 - python
 - 记账
 - pandas
top_img: /image/16.jpg

---


项目实施方案

## 一、 整体方案概述

由于花钱太快，需要开发一款名为 “账本助手 (Ledger Assistant)” 的桌面端工具。该工具将基于 Python 语言开发，以 PySide6 构建图形化界面，以 SQLite 作为本地数据存储引擎，确保数据安全与处理性能。项目将采用模块化设计，核心功能包括数据导入、清洗、分类、存储、可视化与导出，确保代码高可维护性与可扩展性。

针对需求中的“自动抓取账单”功能，经技术可行性评估，官方 API 申请门槛高（主要面向商户），模拟登录存在法律风险且微信/支付宝风控严格，极易导致账号被封禁。为确保项目的稳定性、安全性及交付质量，本方案将不实现“自动抓取账单”功能，专注于优化“官方导出文件导入”这一核心流程，确保其体验流畅、稳定可靠。

## 二、 技术选型与架构

模块	技术选型	说明

|模块|技术选型|说明|
| ------| ----------| ------------------------------------------------------------------------------------|
|**图形界面 (GUI)**|**PySide6**|相比Tkinter，界面更美观、组件更丰富，可实现更现代的用户体验，便于后期功能扩展。|
|**数据处理**|**pandas**|行业标准库，强大且高效，完美处理CSV/Excel数据的读取、清洗、合并与转换。|
|**数据存储**|**SQLite**|轻量级本地数据库，性能优于CSV的直接读写，支持SQL查询，便于数据管理与增量导入。|
|**数据可视化**|**matplotlib + seaborn**|静态图表生成稳定可靠，满足所有报表需求，并能方便地嵌入到PySide6的窗口中。|
|**报表导出**|**matplotlib.backends** + **Pillow**|将图表保存为图片，结合数据，可生成包含图表和表格的PDF/图片报告。|
|**打包与分发**|**PyInstaller**|成熟稳定，支持Windows和macOS单文件打包，方便用户直接双击运行，无需安装Python环境。|
|**日志与异常**|**logging**|Python标准库，记录关键操作和错误信息，便于问题追踪。|

## 三、 功能实现详解

### 3.1 数据获取与清洗模块

文件导入：支持拖拽或文件选择的方式，导入微信/支付宝官方导出的 CSV 或 Excel 文件。将提供字段映射配置界面，允许用户根据未来文件格式变化进行调整，确保兼容性。我们提供一个独立的数据库模块 `database.py`，负责 SQLite 建表、数据入库（含增量导入）、查询加载等功能。

```python
"""
database.py

SQLite 数据库操作模块，提供建表、增量插入、查询、更新分类等功能。
"""

import sqlite3
import pandas as pd
import os
from typing import Optional, List


class LedgerDatabase:
    """账本数据库操作类"""

    def __init__(self, db_path: str = "./data/ledger.db"):
        self.db_path = db_path
        os.makedirs(os.path.dirname(db_path), exist_ok=True)
        self._init_tables()

    def _init_tables(self):
        """初始化数据库表结构"""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS transactions (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    transaction_time TEXT NOT NULL,
                    counterparty TEXT NOT NULL,
                    description TEXT NOT NULL,
                    amount REAL NOT NULL,
                    type TEXT NOT NULL,
                    payment_method TEXT,
                    status TEXT,
                    category TEXT DEFAULT '未分类',
                    user_modified INTEGER DEFAULT 0,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            ''')
            # 创建索引
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_time ON transactions (transaction_time)')
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_type ON transactions (type)')
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_category ON transactions (category)')
            cursor.execute('''
                CREATE INDEX IF NOT EXISTS idx_unique_check ON transactions
                (transaction_time, amount, counterparty, description)
            ''')
            conn.commit()

    def insert_transactions(self, df: pd.DataFrame) -> int:
        """
        将 DataFrame 中的交易记录插入数据库，自动过滤已存在的记录。
        DataFrame 中应包含 'category' 列（若没有则默认为'未分类'）。
        返回新增的记录数。
        """
        if df.empty:
            return 0

        new_count = 0
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            for _, row in df.iterrows():
                # 检查是否已存在
                cursor.execute('''
                    SELECT id FROM transactions
                    WHERE transaction_time = ? AND amount = ? AND counterparty = ? AND description = ?
                ''', (
                    row['transaction_time'].isoformat() if hasattr(row['transaction_time'], 'isoformat') else row['transaction_time'],
                    row['amount'],
                    row['counterparty'],
                    row['description']
                ))
                if cursor.fetchone() is None:
                    category = row.get('category', '未分类')
                    cursor.execute('''
                        INSERT INTO transactions
                        (transaction_time, counterparty, description, amount, type, payment_method, status, category)
                        VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                    ''', (
                        row['transaction_time'].isoformat() if hasattr(row['transaction_time'], 'isoformat') else row['transaction_time'],
                        row['counterparty'],
                        row['description'],
                        row['amount'],
                        row['type'],
                        row.get('payment_method', ''),
                        row.get('status', ''),
                        category
                    ))
                    new_count += 1
            conn.commit()
        return new_count

    def load_all(self) -> pd.DataFrame:
        """加载所有交易记录"""
        with sqlite3.connect(self.db_path) as conn:
            df = pd.read_sql_query("SELECT * FROM transactions ORDER BY transaction_time DESC", conn)
        if not df.empty and 'transaction_time' in df.columns:
            df['transaction_time'] = pd.to_datetime(df['transaction_time'])
        return df

    def load_by_date_range(self, start_date: str, end_date: str) -> pd.DataFrame:
        """按日期范围加载交易记录（日期格式 YYYY-MM-DD）"""
        with sqlite3.connect(self.db_path) as conn:
            df = pd.read_sql_query('''
                SELECT * FROM transactions
                WHERE transaction_time BETWEEN ? AND ?
                ORDER BY transaction_time DESC
            ''', conn, params=(start_date, end_date))
        if not df.empty and 'transaction_time' in df.columns:
            df['transaction_time'] = pd.to_datetime(df['transaction_time'])
        return df

    def get_record_count(self) -> int:
        """获取总记录数"""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            cursor.execute("SELECT COUNT(*) FROM transactions")
            return cursor.fetchone()[0]

    def update_category(self, transaction_id: int, category: str):
        """更新某条交易的分类，并标记为用户手动修改"""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            cursor.execute('''
                UPDATE transactions
                SET category = ?, user_modified = 1
                WHERE id = ?
            ''', (category, transaction_id))
            conn.commit()

    def get_distinct_counterparties(self, limit: int = 10) -> List[str]:
        """获取高频交易对方 Top N"""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            cursor.execute('''
                SELECT counterparty, COUNT(*) as cnt
                FROM transactions
                GROUP BY counterparty
                ORDER BY cnt DESC
                LIMIT ?
            ''', (limit,))
            return [row[0] for row in cursor.fetchall()]

    def export_to_excel(self, filepath: str, start_date: Optional[str] = None, end_date: Optional[str] = None):
        """导出数据到 Excel 文件"""
        if start_date and end_date:
            df = self.load_by_date_range(start_date, end_date)
        else:
            df = self.load_all()
        if not df.empty:
            df.to_excel(filepath, index=False, engine='openpyxl')
        else:
            raise ValueError("没有数据可导出")

    def clear_all(self):
        """清空所有数据（谨慎使用）"""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            cursor.execute("DELETE FROM transactions")
            conn.commit()
```

‍

我们建议将数据处理逻辑独立到一个新文件中，例如 `data_processor.py`​，以便于代码维护、测试和未来扩展。GUI 文件（如 `main_window.py`​）只负责界面交互，调用 `data_processor` 中的函数完成业务逻辑。

以下先给出 `data_processor.py` 的完整实现，然后在 GUI 中如何调用的示例。

```python

import re
import pandas as pd
from typing import List, Optional
#微信账单字段映射（以官方导出CSV为例）
WECHAT_FIELD_MAP = {
    '交易时间': 'transaction_time',
    '交易类型': 'transaction_type_raw',
    '交易对方': 'counterparty',
    '商品': 'description',
    '金额(元)': 'amount_raw',          # Excel 中列名为“金额(元)”
    '收/支': 'income_expense',        # 有些版本有“收/支”列
    '支付方式': 'payment_method',
    '当前状态': 'status'
}
#支付宝账单字段映射（以官方导出CSV为例）
ALIPAY_FIELD_MAP = {
    '交易时间': 'transaction_time',
    '交易分类': 'transaction_type_raw',
    '交易对方': 'counterparty',
    '商品说明': 'description',
    '金额': 'amount_raw',
    '收/支': 'income_expense',
    '交易状态': 'status'
}
#平台识别关键词（用于自动判断文件类型）
PLATFORM_KEYWORDS = {
    '微信': ['微信支付', '交易时间', '交易对方', '商品'],
    '支付宝': ['支付宝', '交易时间', '交易对方', '商品说明']
}
def detect_encoding(file_path: str) -> str:
    """尝试多种编码读取CSV，返回可用的编码"""
    encodings = ['utf-8-sig', 'gbk', 'gb2312', 'utf-8']
    for enc in encodings:
        try:
            with open(file_path, 'r', encoding=enc) as f:
                f.read(1000)  # 读取部分内容测试
            return enc
        except UnicodeDecodeError:
            continue
    return 'utf-8-sig'  # 默认
def find_data_start_row(file_path: str) -> int:
    """
    查找 Excel 文件中的数据起始行（表头所在行）。
    返回行索引（0-based），如果未找到则返回0。
    """
    # 读取所有行，不解析为 DataFrame
    df_raw = pd.read_excel(file_path, header=None, dtype=str)
    for i, row in df_raw.iterrows():
        # 将行转换为字符串，检查是否包含关键列名
        row_text = ' '.join(row.astype(str))
        if '交易时间' in row_text and '交易对方' in row_text and '金额' in row_text:
            return i
    return 0
def clean_amount(amount_str) -> float:
    """
    #将金额字符串转换为浮点数，去除货币符号和千位分隔符。
    例如：'¥10.00' -> 10.00, '-5.00' -> -5.00
    """
    if pd.isna(amount_str):
        return 0.0
    # 去除货币符号（¥、￥、$等）和逗号
    cleaned = re.sub(r'[¥￥$,\s]', '', str(amount_str))
    # 处理括号表示负数，如 (10.00) -> -10.00
    if cleaned.startswith('(') and cleaned.endswith(')'):
        cleaned = '-' + cleaned[1:-1]
    try:
        return float(cleaned)
    except ValueError:
        return 0.0
def normalize_transaction_type(df: pd.DataFrame, platform: str) -> pd.DataFrame:
    """根据平台和原始字段，统一生成标准交易类型字段 'type'（income/expense）"""
    def get_type(row):
        if platform == '微信':
            # 优先使用“收/支”列（如果存在）
            if 'income_expense' in df.columns:
                ie = row.get('income_expense', '')
                if '收入' in ie:
                    return 'income'
                elif '支出' in ie:
                    return 'expense'
            # 否则根据“交易类型”列判断
            raw = row.get('transaction_type_raw', '')
            if '收入' in raw:
                return 'income'
            elif '支出' in raw:
                return 'expense'
            else:
                return 'income' if row['amount'] > 0 else 'expense'
        elif platform == '支付宝':
            income_expense = row.get('income_expense', '')
            if '收入' in income_expense:
                return 'income'
            elif '支出' in income_expense:
                return 'expense'
            else:
                return 'income' if row['amount'] > 0 else 'expense'
        else:
            return 'expense' if row['amount'] < 0 else 'income'
    df['type'] = df.apply(get_type, axis=1)
    return df
def filter_invalid_records(df: pd.DataFrame) -> pd.DataFrame:
    """
    过滤无效记录：交易状态非成功、包含退款、转账到本人等。
    """
    # 保留的状态关键词
    valid_status_keywords = ['成功', '支付成功', '交易成功']
    if 'status' in df.columns:
        mask = df['status'].str.contains('|'.join(valid_status_keywords), na=False)
        df = df[mask].copy()
    # 排除包含退款、转账到本人的记录
    if 'description' in df.columns:
        exclude_keywords = ['退款', '转账到本人']
        mask = ~df['description'].str.contains('|'.join(exclude_keywords), na=False)
        df = df[mask].copy()
    return df
def apply_custom_filters(df: pd.DataFrame, keywords: Optional[List[str]] = None) -> pd.DataFrame:
    """
    应用用户自定义过滤规则（如剔除“零钱提现”）。
    """
    if keywords:
        pattern = '|'.join(keywords)
        mask = ~(df['description'].str.contains(pattern, na=False) |
                 df['counterparty'].str.contains(pattern, na=False))
        df = df[mask].copy()
    return df
def load_and_clean(file_path: str,
                   custom_filter_keywords: Optional[List[str]] = None) -> pd.DataFrame:
    """
    加载并清洗单个账单文件，返回标准化后的 DataFrame。
    """
    # 1. 读取文件
    try:
        if file_path.endswith('.csv'):
            encoding = detect_encoding(file_path)
            df = pd.read_csv(file_path, encoding=encoding)
        elif file_path.endswith(('.xlsx', '.xls')):
            # 检查 openpyxl 是否可用
            try:
                import openpyxl
            except ImportError:
                raise ImportError("需要安装 openpyxl 以支持 Excel 文件：pip install openpyxl")
            # 查找数据起始行
            start_row = find_data_start_row(file_path)
            # 重新读取，跳过前导行，并使用第一行作为列名
            df = pd.read_excel(file_path, header=start_row, dtype=str)
            # 清理列名中的空格
            df.columns = [col.strip() if isinstance(col, str) else col for col in df.columns]
        else:
            raise ValueError("不支持的文件格式，请提供 CSV 或 Excel 文件")
    except Exception as e:
        raise RuntimeError(f"文件读取失败: {e}")
    # 2. 自动识别平台（使用列名和少量数据内容）
    platform = None
    all_text = ' '.join(df.columns) + ' ' + ' '.join(df.head(3).astype(str).values.flatten())
    for p, keywords in PLATFORM_KEYWORDS.items():
        if any(keyword in all_text for keyword in keywords):
            platform = p
            break
    if platform is None:
        raise ValueError("无法识别平台，请确保文件是微信或支付宝官方导出的账单")
    # 3. 字段映射
    field_map = WECHAT_FIELD_MAP if platform == '微信' else ALIPAY_FIELD_MAP
    existing_columns = [col for col in field_map.keys() if col in df.columns]
    if not existing_columns:
        raise ValueError("文件字段不匹配，无法映射")
    df = df[existing_columns].rename(columns=field_map)
    # 4. 清洗金额
    df['amount'] = df['amount_raw'].apply(clean_amount)
    # 如果存在“收/支”列且金额尚未反映收支，则调整符号
    if platform == '微信' and 'income_expense' in df.columns:
        df['amount'] = df.apply(
            lambda row: abs(row['amount']) if row['income_expense'] == '收入' else -abs(row['amount']),
            axis=1
        )
    elif platform == '支付宝' and 'income_expense' in df.columns:
        df['amount'] = df.apply(
            lambda row: abs(row['amount']) if row['income_expense'] == '收入' else -abs(row['amount']),
            axis=1
        )
    # 5. 标准化交易类型
    df = normalize_transaction_type(df, platform)
    # 6. 过滤无效记录
    df = filter_invalid_records(df)
    # 7. 应用自定义过滤规则
    if custom_filter_keywords:
        df = apply_custom_filters(df, custom_filter_keywords)
    # 8. 删除临时列
    df.drop(columns=['amount_raw'], inplace=True, errors='ignore')
    if 'income_expense' in df.columns:
        df.drop(columns=['income_expense'], inplace=True)
    if 'transaction_type_raw' in df.columns:
        df.drop(columns=['transaction_type_raw'], inplace=True)
    # 9. 确保必要字段存在
    required_columns = ['transaction_time', 'counterparty', 'description', 'amount',
                        'type', 'payment_method', 'status']
    for col in required_columns:
        if col not in df.columns:
            df[col] = ''
    # 10. 转换交易时间为 datetime
    df['transaction_time'] = pd.to_datetime(df['transaction_time'], errors='coerce')
    return df
def merge_and_deduplicate(df_list: List[pd.DataFrame]) -> pd.DataFrame:
    """
    合并多个 DataFrame 并去重（基于交易时间、金额、对方、描述组合）。
    """
    if not df_list:
        return pd.DataFrame()
    combined = pd.concat(df_list, ignore_index=True)
    combined = combined.drop_duplicates(
        subset=['transaction_time', 'amount', 'counterparty', 'description'],
        keep='first'
    )
    return combined
```

‍

### 3.2 分类与标签模块

我们将在现有项目中增加分类与标签模块，包括内置规则、自定义规则UI和手动分类UI。以下是实现步骤和代码。

#### 一、新增 `category_manager.py`：管理分类规则

```python
"""
category_manager.py

管理分类规则：加载/保存内置规则和用户自定义规则，提供分类匹配功能。
"""

import json
import os
from typing import Dict, List, Optional


class CategoryManager:
    """分类规则管理器"""

    DEFAULT_RULES = {
        # 关键词 -> 分类
        "星巴克": "餐饮",
        "瑞幸": "餐饮",
        "麦当劳": "餐饮",
        "肯德基": "餐饮",
        "蜜雪冰城": "餐饮",
        "美团": "餐饮",
        "饿了么": "餐饮",
        "滴滴": "交通",
        "高德打车": "交通",
        "深圳通": "交通",
        "广州地铁": "交通",
        "美团单车": "交通",
        "京东": "购物",
        "淘宝": "购物",
        "拼多多": "购物",
        "Apple": "购物",
        "网易云音乐": "娱乐",
        "腾讯视频": "娱乐",
        "Steam": "娱乐",
        "工资": "收入",
        "转账": "转账",
        "退款": "退款",  # 但会被过滤掉，这里仅示例
        "零钱提现": "转账",
        "零钱充值": "转账",
        "提现": "转账"
    }

    def __init__(self, config_path: str = "./data/category_rules.json"):
        self.config_path = config_path
        self.rules = self._load_rules()

    def _load_rules(self) -> Dict[str, str]:
        """加载规则，优先使用用户自定义规则，否则使用默认规则"""
        if os.path.exists(self.config_path):
            try:
                with open(self.config_path, 'r', encoding='utf-8') as f:
                    rules = json.load(f)
                return rules
            except Exception:
                pass
        # 返回默认规则的副本
        return self.DEFAULT_RULES.copy()

    def save_rules(self):
        """保存当前规则到文件"""
        os.makedirs(os.path.dirname(self.config_path), exist_ok=True)
        with open(self.config_path, 'w', encoding='utf-8') as f:
            json.dump(self.rules, f, ensure_ascii=False, indent=2)

    def add_rule(self, keyword: str, category: str):
        """添加或更新规则"""
        self.rules[keyword] = category
        self.save_rules()

    def remove_rule(self, keyword: str):
        """删除规则"""
        if keyword in self.rules:
            del self.rules[keyword]
            self.save_rules()

    def get_all_rules(self) -> Dict[str, str]:
        """获取所有规则"""
        return self.rules.copy()

    def classify(self, description: str, counterparty: str) -> str:
        """
        根据商品说明和交易对方返回匹配的分类。
        返回分类名称，若无匹配则返回 "未分类"。
        """
        # 合并搜索文本
        search_text = f"{description} {counterparty}"
        for keyword, category in self.rules.items():
            if keyword in search_text:
                return category
        return "未分类"
```

内置规则库：提供一份内置关键词映射表（例如 {"星巴克": "餐饮", "滴滴出行": "交通"}）。

自定义规则：用户可通过 GUI 界面添加、修改、删除关键词映射规则。规则将以 JSON 格式存储，方便用户备份和共享。

#### 二、新增 `category_rule_dialog.py`：分类规则配置对话框

```python
"""
category_rule_dialog.py

分类规则设置对话框，用于查看、添加、修改、删除关键词映射。
"""

from PySide6.QtWidgets import (
    QDialog, QVBoxLayout, QHBoxLayout, QTableWidget, QTableWidgetItem,
    QPushButton, QMessageBox, QLineEdit, QComboBox, QHeaderView
)
from PySide6.QtCore import Qt, Signal


class CategoryRuleDialog(QDialog):
    """分类规则编辑对话框"""
    # 定义信号，当规则变化时通知主窗口刷新
    rules_changed = Signal()

    def __init__(self, category_manager, parent=None):
        super().__init__(parent)
        self.category_manager = category_manager
        self.setWindowTitle("分类规则设置")
        self.resize(600, 400)

        layout = QVBoxLayout(self)

        # 表格展示规则
        self.table = QTableWidget()
        self.table.setColumnCount(2)
        self.table.setHorizontalHeaderLabels(["关键词", "分类"])
        self.table.horizontalHeader().setSectionResizeMode(QHeaderView.Stretch)
        layout.addWidget(self.table)

        # 按钮区域
        btn_layout = QHBoxLayout()

        self.add_btn = QPushButton("添加")
        self.edit_btn = QPushButton("编辑")
        self.delete_btn = QPushButton("删除")
        self.close_btn = QPushButton("关闭")

        btn_layout.addWidget(self.add_btn)
        btn_layout.addWidget(self.edit_btn)
        btn_layout.addWidget(self.delete_btn)
        btn_layout.addStretch()
        btn_layout.addWidget(self.close_btn)

        layout.addLayout(btn_layout)

        # 连接信号
        self.add_btn.clicked.connect(self.add_rule)
        self.edit_btn.clicked.connect(self.edit_rule)
        self.delete_btn.clicked.connect(self.delete_rule)
        self.close_btn.clicked.connect(self.accept)

        self.refresh_table()

    def refresh_table(self):
        """刷新表格显示当前所有规则"""
        rules = self.category_manager.get_all_rules()
        self.table.setRowCount(len(rules))
        for i, (keyword, category) in enumerate(rules.items()):
            self.table.setItem(i, 0, QTableWidgetItem(keyword))
            self.table.setItem(i, 1, QTableWidgetItem(category))

    def add_rule(self):
        """添加新规则"""
        keyword, category = self._get_input("添加规则", "请输入关键词和分类")
        if keyword and category:
            self.category_manager.add_rule(keyword, category)
            self.refresh_table()
            self.rules_changed.emit()

    def edit_rule(self):
        """编辑当前选中的规则"""
        current_row = self.table.currentRow()
        if current_row < 0:
            QMessageBox.warning(self, "提示", "请先选择要编辑的规则。")
            return
        old_keyword = self.table.item(current_row, 0).text()
        old_category = self.table.item(current_row, 1).text()
        keyword, category = self._get_input("编辑规则", "修改关键词和分类", old_keyword, old_category)
        if keyword and category:
            # 删除旧的，添加新的
            self.category_manager.remove_rule(old_keyword)
            self.category_manager.add_rule(keyword, category)
            self.refresh_table()
            self.rules_changed.emit()

    def delete_rule(self):
        current_row = self.table.currentRow()
        if current_row < 0:
            QMessageBox.warning(self, "提示", "请先选择要删除的规则。")
            return
        keyword = self.table.item(current_row, 0).text()
        reply = QMessageBox.question(self, "确认删除", f"确定要删除规则 '{keyword}' 吗？",
                                     QMessageBox.Yes | QMessageBox.No)
        if reply == QMessageBox.Yes:
            self.category_manager.remove_rule(keyword)
            self.refresh_table()
            self.rules_changed.emit()

    def _get_input(self, title: str, label: str, default_keyword: str = "", default_category: str = "") -> tuple:
        """弹出对话框获取关键词和分类"""
        from PySide6.QtWidgets import QInputDialog
        keyword, ok1 = QInputDialog.getText(self, title, f"{label}\n关键词:", text=default_keyword)
        if not ok1 or not keyword.strip():
            return None, None
        categories = ["餐饮", "交通", "购物", "娱乐", "住房", "收入", "转账", "其他", "未分类"]
        category, ok2 = QInputDialog.getItem(self, title, f"{label}\n分类:", categories, 0, False)
        if not ok2:
            return None, None
        return keyword.strip(), category
```
您的分类规则能够被记住，因为程序在背后自动将它们保存到了本地文件中，并在每次启动时重新加载。具体来说：

1. **保存位置**  
   您通过“配置”→“分类规则设置”添加或修改的所有规则，都会被写入项目目录下的 `data/category_rules.json` 文件。这是一个纯文本的 JSON 文件，里面存储着“关键词 → 分类”的映射关系。

2. **保存时机**  
   每当您在规则管理对话框中点击“添加”、“编辑”或“删除”后，程序会立即调用 `CategoryManager` 的 `save_rules()` 方法，将当前内存中的所有规则写入 `category_rules.json` 文件。因此规则不会丢失。

3. **加载时机**  
   每次启动程序时，`CategoryManager` 的 `__init__` 方法会检查 `data/category_rules.json` 是否存在。如果存在，就读取该文件并加载规则；如果不存在（首次运行），则使用代码中内置的默认规则。

4. **内置规则与用户规则的关系**  
   - 内置规则写在 `category_manager.py` 的 `DEFAULT_RULES` 字典中，作为后备。  
   - 一旦您添加或修改了任何规则，程序会优先使用 `category_rules.json` 中的用户规则，并且用户规则会覆盖同关键词的内置规则。  
   - 如果您从未添加过任何规则，则始终使用内置规则，且不会生成 `category_rules.json` 文件。

5. **为什么不需要手动保存？**  
   因为程序已经封装好了持久化逻辑，您只需要通过图形界面操作，后台会自动处理文件的读写。这种设计让用户体验更流畅，无需关心底层存储细节。

所以，下次您打开程序，之前辛辛苦苦添加的所有关键词映射都会原样出现，直接生效。您也可以随时备份 `data/category_rules.json` 文件，以防意外丢失。


### 3.3 数据存储模块

增量导入：系统会检查数据库中已存在的交易记录（基于“交易时间 + 交易金额 + 交易对方”的唯一组合）。导入时，新文件中的记录若在数据库中不存在，则插入；若已存在，则跳过，避免重复。

本地存储：所有数据存储在用户目录下的"D:\python\bill\data\ledger.db"文件中，确保数据完全本地化。

### 3.4 可视化与报表模块

接下来完成所有可视化报表（折线图、饼图、热力图等）的生成与嵌入

```python
"""
report_generator.py

生成各种报表图表，返回 matplotlib Figure 对象。
优化样式，解决文字重叠问题；饼图自动合并占比过小的分类。
"""

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
from matplotlib.figure import Figure
import seaborn as sns
import calendar

# 设置全局样式
sns.set_style("whitegrid")
plt.rcParams['font.sans-serif'] = ['SimHei', 'Microsoft YaHei', 'WenQuanYi Zen Hei', 'DejaVu Sans']
plt.rcParams['axes.unicode_minus'] = False
plt.rcParams['figure.dpi'] = 100
plt.rcParams['savefig.dpi'] = 150
plt.rcParams['figure.autolayout'] = True
plt.rcParams['lines.linewidth'] = 2
plt.rcParams['lines.markersize'] = 6


def plot_income_expense_trend(df: pd.DataFrame, start_date: str, end_date: str) -> Figure:
    """
    收支趋势折线图
    """
    if df.empty:
        fig = Figure(figsize=(10, 5))
        ax = fig.add_subplot(111)
        ax.text(0.5, 0.5, "无数据", ha='center', va='center', fontsize=14)
        ax.set_title("收支趋势")
        return fig

    # 按日期分组
    df_daily = df.copy()
    df_daily['date'] = df_daily['transaction_time'].dt.date
    income = df_daily[df_daily['type'] == 'income'].groupby('date')['amount'].sum()
    expense = df_daily[df_daily['type'] == 'expense'].groupby('date')['amount'].sum().abs()
    trend = pd.DataFrame({'收入': income, '支出': expense}).fillna(0).sort_index()

    fig = Figure(figsize=(10, 5))
    ax = fig.add_subplot(111)
    ax.plot(trend.index, trend['收入'], marker='o', label='收入', color='#2E8B57', linewidth=2, markersize=4)
    ax.plot(trend.index, trend['支出'], marker='s', label='支出', color='#CD5C5C', linewidth=2, markersize=4)
    ax.set_title(f'收支趋势 ({start_date} 至 {end_date})', fontsize=14, fontweight='bold')
    ax.set_xlabel('日期', fontsize=11)
    ax.set_ylabel('金额 (元)', fontsize=11)
    ax.legend(loc='upper left', fontsize=10)
    ax.grid(True, linestyle='--', alpha=0.6)
    # 自动旋转日期标签
    fig.autofmt_xdate(rotation=30)
    # 格式化日期显示
    ax.xaxis.set_major_formatter(mdates.DateFormatter('%m-%d'))
    fig.tight_layout()
    return fig


def plot_category_pie(df: pd.DataFrame, min_percent: float = 3.0) -> Figure:
    """
    分类支出占比饼图（只统计支出）
    自动合并占比小于 min_percent 的分类为“其他”，使图表更清晰。

    参数:
        df: 包含交易数据的 DataFrame
        min_percent: 合并阈值（百分比），默认 3%
    """
    if df.empty:
        fig = Figure(figsize=(8, 6))
        ax = fig.add_subplot(111)
        ax.text(0.5, 0.5, "无数据", ha='center', va='center', fontsize=14)
        ax.set_title("分类支出占比")
        return fig

    expense_df = df[df['type'] == 'expense']
    if expense_df.empty:
        fig = Figure(figsize=(8, 6))
        ax = fig.add_subplot(111)
        ax.text(0.5, 0.5, "无支出数据", ha='center', va='center', fontsize=14)
        ax.set_title("分类支出占比")
        return fig

    category_sum = expense_df.groupby('category')['amount'].sum().abs()
    category_sum = category_sum[category_sum > 0]
    if category_sum.empty:
        fig = Figure(figsize=(8, 6))
        ax = fig.add_subplot(111)
        ax.text(0.5, 0.5, "无有效分类数据", ha='center', va='center', fontsize=14)
        ax.set_title("分类支出占比")
        return fig

    total = category_sum.sum()
    # 计算百分比
    percentages = (category_sum / total) * 100
    # 合并小于阈值的分类
    small_cats = percentages[percentages < min_percent].index
    if len(small_cats) > 0:
        other_sum = category_sum[small_cats].sum()
        # 保留大分类
        main_cats = category_sum[~category_sum.index.isin(small_cats)]
        if other_sum > 0:
            main_cats['其他'] = other_sum
        category_sum = main_cats

    # 重新排序（按金额降序）
    category_sum = category_sum.sort_values(ascending=False)

    fig = Figure(figsize=(9, 6))
    ax = fig.add_subplot(111)
    # 圆环效果
    wedges, texts, autotexts = ax.pie(category_sum, labels=None, autopct='%1.1f%%',
                                      startangle=90, pctdistance=0.85,
                                      wedgeprops=dict(width=0.6, edgecolor='white'))
    # 添加图例（右侧）
    ax.legend(wedges, category_sum.index, title="分类", loc="center left", bbox_to_anchor=(1, 0, 0.5, 1))
    # 调整百分比文字样式
    for autotext in autotexts:
        autotext.set_fontsize(9)
        autotext.set_color('white')
    ax.set_title('分类支出占比', fontsize=14, fontweight='bold')
    fig.tight_layout()
    return fig


def plot_calendar_heatmap(df: pd.DataFrame, year: int, month: int) -> Figure:
    """
    收支日历热力图（展示每日支出金额）
    """
    if df.empty:
        fig = Figure(figsize=(10, 4))
        ax = fig.add_subplot(111)
        ax.text(0.5, 0.5, "无数据", ha='center', va='center', fontsize=14)
        ax.set_title(f"{year}年{month}月 消费热力图")
        return fig

    start = f"{year}-{month:02d}-01"
    if month == 12:
        end = f"{year+1}-01-01"
    else:
        end = f"{year}-{month+1:02d}-01"
    month_df = df[(df['transaction_time'] >= start) & (df['transaction_time'] < end) & (df['type'] == 'expense')]
    if month_df.empty:
        fig = Figure(figsize=(10, 4))
        ax = fig.add_subplot(111)
        ax.text(0.5, 0.5, "当月无支出", ha='center', va='center', fontsize=14)
        ax.set_title(f"{year}年{month}月 消费热力图")
        return fig

    days_in_month = calendar.monthrange(year, month)[1]
    date_list = pd.date_range(start=start, periods=days_in_month, freq='D')
    daily_expense = month_df.groupby(month_df['transaction_time'].dt.date)['amount'].sum().abs()

    weeks = []
    week_days = []
    for dt in date_list:
        week_num = dt.isocalendar()[1]
        week_day = dt.weekday()
        weeks.append(week_num)
        week_days.append(week_day)

    min_week = min(weeks)
    max_week = max(weeks)
    n_weeks = max_week - min_week + 1
    heatmap_data = np.zeros((7, n_weeks))
    for dt, week_num, week_day in zip(date_list, weeks, week_days):
        amount = daily_expense.get(dt.date(), 0)
        col_idx = week_num - min_week
        heatmap_data[week_day, col_idx] = amount

    fig = Figure(figsize=(11, 4))
    ax = fig.add_subplot(111)
    sns.heatmap(heatmap_data, annot=True, fmt='.0f', cmap='YlOrRd',
                cbar_kws={'label': '支出金额(元)', 'shrink': 0.8},
                ax=ax, annot_kws={'size': 8})
    ax.set_xticklabels([f'第{i+1}周' for i in range(n_weeks)], rotation=45, fontsize=9)
    ax.set_yticklabels(['周一', '周二', '周三', '周四', '周五', '周六', '周日'], rotation=0, fontsize=9)
    ax.set_title(f"{year}年{month}月 消费热力图", fontsize=14, fontweight='bold')
    fig.tight_layout()
    return fig


def plot_income_sources(df: pd.DataFrame) -> Figure:
    """
    收入来源分析（按对方分组求和）水平条形图
    """
    if df.empty:
        fig = Figure(figsize=(8, 5))
        ax = fig.add_subplot(111)
        ax.text(0.5, 0.5, "无数据", ha='center', va='center', fontsize=14)
        ax.set_title("收入来源")
        return fig

    income_df = df[df['type'] == 'income']
    if income_df.empty:
        fig = Figure(figsize=(8, 5))
        ax = fig.add_subplot(111)
        ax.text(0.5, 0.5, "无收入数据", ha='center', va='center', fontsize=14)
        ax.set_title("收入来源")
        return fig

    source_sum = income_df.groupby('counterparty')['amount'].sum().sort_values(ascending=True).tail(10)
    if source_sum.empty:
        fig = Figure(figsize=(8, 5))
        ax = fig.add_subplot(111)
        ax.text(0.5, 0.5, "无有效数据", ha='center', va='center', fontsize=14)
        ax.set_title("收入来源")
        return fig

    fig = Figure(figsize=(8, 6))
    ax = fig.add_subplot(111)
    bars = ax.barh(source_sum.index, source_sum.values, color='#2E8B57', alpha=0.8)
    ax.set_xlabel('金额 (元)', fontsize=11)
    ax.set_title('收入来源 Top 10', fontsize=14, fontweight='bold')
    for bar in bars:
        width = bar.get_width()
        ax.text(width + max(source_sum.values)*0.01, bar.get_y() + bar.get_height()/2,
                f'{width:.0f}', va='center', fontsize=9)
    ax.grid(axis='x', linestyle='--', alpha=0.6)
    fig.tight_layout()
    return fig


def plot_top_counterparties(df: pd.DataFrame, top_n: int = 10) -> Figure:
    """
    高频交易对方 TopN（按交易次数，支出+收入）
    """
    if df.empty:
        fig = Figure(figsize=(8, 5))
        ax = fig.add_subplot(111)
        ax.text(0.5, 0.5, "无数据", ha='center', va='center', fontsize=14)
        ax.set_title("高频交易对方")
        return fig

    counterparty_counts = df.groupby('counterparty').size().sort_values(ascending=True).tail(top_n)
    if counterparty_counts.empty:
        fig = Figure(figsize=(8, 5))
        ax = fig.add_subplot(111)
        ax.text(0.5, 0.5, "无有效数据", ha='center', va='center', fontsize=14)
        ax.set_title("高频交易对方")
        return fig

    fig = Figure(figsize=(9, 6))
    ax = fig.add_subplot(111)
    bars = ax.barh(counterparty_counts.index, counterparty_counts.values, color='#4682B4', alpha=0.8)
    ax.set_xlabel('交易次数', fontsize=11)
    ax.set_title(f'高频交易对方 Top {top_n}', fontsize=14, fontweight='bold')
    for bar in bars:
        width = bar.get_width()
        ax.text(width + 0.2, bar.get_y() + bar.get_height()/2,
                f'{int(width)}', va='center', fontsize=9)
    ax.grid(axis='x', linestyle='--', alpha=0.6)
    fig.tight_layout()
    return fig
```

交互式报表界面：在主界面中，用户可以通过选择时间范围（月/年/自定义）来动态刷新所有报表。

‍

最后，我们来实现报表导出功能：将当前时间范围内的所有图表（折线图、饼图、热力图、收入来源条形图、高频交易方条形图）导出为一个 PDF 文件，同时支持导出为单独的图片文件

```python
"""
report_exporter.py

报表导出模块：将当前 matplotlib Figure 对象导出为 PDF 或图片。
"""

import os
from matplotlib.backends.backend_pdf import PdfPages
import matplotlib.pyplot as plt


def export_to_pdf(figures, filepath: str):
    """
    将多个 matplotlib Figure 对象保存到一个 PDF 文件中。
    figures: 列表，元素为 (figure, title) 元组，title 用于可选的页面标题。
    """
    with PdfPages(filepath) as pdf:
        for fig, title in figures:
            # 可选：在每页顶部添加标题
            fig.suptitle(title, fontsize=14, weight='bold')
            pdf.savefig(fig, bbox_inches='tight')
            # 清除标题，避免影响后续使用
            fig.suptitle('')
    return filepath


def export_to_images(figures, output_dir: str, dpi=150):
    """
    将多个 matplotlib Figure 对象保存为图片文件（PNG）。
    figures: 列表，元素为 (figure, filename) 元组，filename 不含扩展名。
    """
    os.makedirs(output_dir, exist_ok=True)
    saved_paths = []
    for fig, filename in figures:
        filepath = os.path.join(output_dir, f"{filename}.png")
        fig.savefig(filepath, dpi=dpi, bbox_inches='tight')
        saved_paths.append(filepath)
    return saved_paths
```

### 3.5 图形化界面设计

界面将采用简洁的布局，主要分为四个区域：

菜单栏：文件导入、规则配置、数据导出、帮助。

工具栏：快速导入、刷新、时间范围选择器。

报表展示区：采用标签页（Tab）形式，分别展示“收支概览”、“分类分析”、“日历与趋势”等。

状态栏：显示当前数据库记录数、上次导入时间、导入状态等信息。

‍

这是整个工作最复杂的一步，不仅需要搭建好ui图式，并且需要联合上述的所有代码共同实现功能

```python
"""
main_window.py

账本助手主窗口，集成 PySide6 GUI，调用数据处理和数据库模块。
支持表格展示交易明细，手动分类，分类规则管理，可视化报表及导出。
"""

import sys
import os
import pandas as pd
import sqlite3
from PySide6.QtWidgets import (
    QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
    QMenuBar, QMenu, QToolBar, QStatusBar, QTabWidget, QLabel,
    QPushButton, QFileDialog, QMessageBox, QComboBox, QDateEdit,
    QTableWidget, QTableWidgetItem, QHeaderView, QInputDialog, QProgressDialog
)
from PySide6.QtCore import Qt, QDate, Slot
from PySide6.QtGui import QAction, QFont

from matplotlib.backends.backend_qt5agg import FigureCanvasQTAgg as FigureCanvas
from matplotlib.figure import Figure

from data_processor import load_and_clean, merge_and_deduplicate
from database import LedgerDatabase
from category_manager import CategoryManager
from category_rule_dialog import CategoryRuleDialog
from report_exporter import export_to_pdf, export_to_images


def resource_path(relative_path):
    """获取资源的绝对路径，兼容开发环境和 PyInstaller 打包后的环境"""
    try:
        base_path = sys._MEIPASS
    except Exception:
        base_path = os.path.abspath(".")
    return os.path.join(base_path, relative_path)


class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("账本助手 - 微信/支付宝账单自动化工具")
        self.resize(1200, 800)

        # 设置窗口图标（可选，请将 icon.ico 放在同目录下）
        # icon_path = resource_path("icon.ico")
        # if os.path.exists(icon_path):
        #     self.setWindowIcon(QIcon(icon_path))

        # 应用全局样式表
        self.setStyleSheet("""
            QMainWindow {
                background-color: #f5f5f5;
            }
            QMenuBar {
                background-color: #ffffff;
                border-bottom: 1px solid #dcdcdc;
            }
            QMenuBar::item {
                padding: 5px 10px;
                spacing: 3px;
            }
            QMenuBar::item:selected {
                background-color: #e6e6e6;
            }
            QToolBar {
                background-color: #ffffff;
                border-bottom: 1px solid #dcdcdc;
                spacing: 5px;
                padding: 5px;
            }
            QPushButton {
                background-color: #4CAF50;
                color: white;
                border: none;
                padding: 6px 12px;
                border-radius: 4px;
                font-size: 12px;
            }
            QPushButton:hover {
                background-color: #45a049;
            }
            QPushButton:pressed {
                background-color: #3e8e41;
            }
            QTabWidget::pane {
                border: 1px solid #dcdcdc;
                background-color: #ffffff;
            }
            QTabBar::tab {
                background-color: #f0f0f0;
                padding: 8px 16px;
                margin-right: 2px;
                border-top-left-radius: 4px;
                border-top-right-radius: 4px;
            }
            QTabBar::tab:selected {
                background-color: #ffffff;
                border-bottom: 2px solid #4CAF50;
            }
            QTabBar::tab:hover {
                background-color: #e6e6e6;
            }
            QTableWidget {
                gridline-color: #e0e0e0;
                selection-background-color: #cce8cc;
                alternate-background-color: #fafafa;
            }
            QHeaderView::section {
                background-color: #f2f2f2;
                padding: 6px;
                border: 1px solid #dcdcdc;
                font-weight: bold;
            }
            QStatusBar {
                background-color: #ffffff;
                border-top: 1px solid #dcdcdc;
            }
            QDateEdit, QComboBox {
                padding: 4px;
                border: 1px solid #cccccc;
                border-radius: 4px;
                background-color: white;
            }
        """)

        # 数据库实例（使用资源路径）
        self.db = LedgerDatabase(resource_path("./data/ledger.db"))
        # 分类管理器（使用资源路径）
        self.category_manager = CategoryManager(resource_path("./data/category_rules.json"))
        # 当前显示的 DataFrame
        self.current_df = pd.DataFrame()
        # 用户自定义过滤规则
        self.custom_filter_keywords = ["零钱提现", "转账到本人"]

        self._create_menu_bar()
        self._create_tool_bar()
        self._create_central_widget()
        self._create_status_bar()
        self._connect_signals()

        self._refresh_data()

    # ------------------ 界面创建 ------------------
    def _create_menu_bar(self):
        menubar = self.menuBar()
        # 文件菜单
        file_menu = menubar.addMenu("文件(&F)")
        self.import_action = QAction("导入账单(&I)...", self)
        self.import_action.setShortcut("Ctrl+I")
        file_menu.addAction(self.import_action)

        self.export_data_action = QAction("导出数据(&E)...", self)
        self.export_data_action.setShortcut("Ctrl+E")
        file_menu.addAction(self.export_data_action)

        file_menu.addSeparator()
        self.exit_action = QAction("退出(&X)", self)
        self.exit_action.setShortcut("Ctrl+Q")
        file_menu.addAction(self.exit_action)

        # 配置菜单
        config_menu = menubar.addMenu("配置(&C)")
        self.category_rule_action = QAction("分类规则设置(&R)...", self)
        config_menu.addAction(self.category_rule_action)
        self.filter_rule_action = QAction("过滤规则设置(&F)...", self)
        config_menu.addAction(self.filter_rule_action)

        # 帮助菜单
        help_menu = menubar.addMenu("帮助(&H)")
        self.user_manual_action = QAction("使用手册(&M)", self)
        help_menu.addAction(self.user_manual_action)
        self.about_action = QAction("关于(&A)", self)
        help_menu.addAction(self.about_action)

    def _create_tool_bar(self):
        toolbar = QToolBar("主工具栏")
        toolbar.setMovable(False)
        self.addToolBar(toolbar)

        self.import_btn = QPushButton("导入账单")
        self.import_btn.setToolTip("导入微信/支付宝账单文件")
        toolbar.addWidget(self.import_btn)

        toolbar.addSeparator()
        self.refresh_btn = QPushButton("刷新报表")
        self.refresh_btn.setToolTip("刷新当前时间范围内的报表")
        toolbar.addWidget(self.refresh_btn)

        toolbar.addSeparator()
        self.export_report_btn = QPushButton("导出报告")
        self.export_report_btn.setToolTip("导出当前报表为PDF或图片")
        toolbar.addWidget(self.export_report_btn)

        toolbar.addSeparator()
        toolbar.addWidget(QLabel("时间范围: "))
        self.date_range_combo = QComboBox()
        self.date_range_combo.addItems(["本月", "上月", "本年度", "自定义"])
        toolbar.addWidget(self.date_range_combo)

        self.start_date_edit = QDateEdit()
        self.start_date_edit.setCalendarPopup(True)
        self.start_date_edit.setDate(QDate.currentDate().addMonths(-1))
        self.start_date_edit.setDisplayFormat("yyyy-MM-dd")
        self.end_date_edit = QDateEdit()
        self.end_date_edit.setCalendarPopup(True)
        self.end_date_edit.setDate(QDate.currentDate())
        self.end_date_edit.setDisplayFormat("yyyy-MM-dd")
        toolbar.addWidget(QLabel("从"))
        toolbar.addWidget(self.start_date_edit)
        toolbar.addWidget(QLabel("到"))
        toolbar.addWidget(self.end_date_edit)

        self.apply_date_btn = QPushButton("应用")
        toolbar.addWidget(self.apply_date_btn)

    def _create_central_widget(self):
        central = QWidget()
        self.setCentralWidget(central)
        layout = QVBoxLayout(central)
        layout.setContentsMargins(5, 5, 5, 5)

        self.tab_widget = QTabWidget()

        # 收支总览页
        self.overview_tab = QWidget()
        overview_layout = QVBoxLayout(self.overview_tab)
        self.overview_canvas = FigureCanvas(Figure(figsize=(10, 5)))
        overview_layout.addWidget(self.overview_canvas)
        self.tab_widget.addTab(self.overview_tab, "收支总览")

        # 分类分析页
        self.category_tab = QWidget()
        category_layout = QVBoxLayout(self.category_tab)
        self.category_canvas = FigureCanvas(Figure(figsize=(8, 6)))
        category_layout.addWidget(self.category_canvas)
        self.tab_widget.addTab(self.category_tab, "分类分析")

        # 日历热力图页
        self.calendar_tab = QWidget()
        calendar_layout = QVBoxLayout(self.calendar_tab)
        self.calendar_canvas = FigureCanvas(Figure(figsize=(10, 4)))
        calendar_layout.addWidget(self.calendar_canvas)
        self.tab_widget.addTab(self.calendar_tab, "日历热力图")

        # 收入来源页
        self.income_tab = QWidget()
        income_layout = QVBoxLayout(self.income_tab)
        self.income_canvas = FigureCanvas(Figure(figsize=(8, 5)))
        income_layout.addWidget(self.income_canvas)
        self.tab_widget.addTab(self.income_tab, "收入来源")

        # 高频交易方页
        self.high_freq_tab = QWidget()
        high_freq_layout = QVBoxLayout(self.high_freq_tab)
        self.high_freq_canvas = FigureCanvas(Figure(figsize=(8, 5)))
        high_freq_layout.addWidget(self.high_freq_canvas)
        self.tab_widget.addTab(self.high_freq_tab, "高频交易方")

        # 交易明细页
        self.detail_tab = QWidget()
        detail_layout = QVBoxLayout(self.detail_tab)
        self.detail_table = QTableWidget()
        self.detail_table.setColumnCount(9)
        self.detail_table.setHorizontalHeaderLabels([
            "ID", "交易时间", "对方", "商品说明", "金额(元)", "类型", "支付方式", "状态", "分类"
        ])
        self.detail_table.setColumnHidden(0, True)
        self.detail_table.horizontalHeader().setSectionResizeMode(QHeaderView.Stretch)
        detail_layout.addWidget(self.detail_table)
        self.tab_widget.addTab(self.detail_tab, "交易明细")

        layout.addWidget(self.tab_widget)

    def _create_status_bar(self):
        self.status_bar = QStatusBar()
        self.setStatusBar(self.status_bar)

        self.status_label = QLabel("就绪")
        self.status_bar.addWidget(self.status_label)

        # 右侧添加永久部件：记录数和署名
        self.record_count_label = QLabel("记录数: 0")
        self.status_bar.addPermanentWidget(self.record_count_label)

        # 知识产权署名
        self.copyright_label = QLabel("made by Gavin")
        self.copyright_label.setStyleSheet("color: #888888; font-style: italic; margin-right: 10px;")
        self.status_bar.addPermanentWidget(self.copyright_label)

        self.last_import_label = QLabel("上次导入: 未导入")
        self.status_bar.addPermanentWidget(self.last_import_label)

    def _connect_signals(self):
        self.import_action.triggered.connect(self.on_import)
        self.export_data_action.triggered.connect(self.on_export_data)
        self.exit_action.triggered.connect(self.close)
        self.category_rule_action.triggered.connect(self.on_category_rules)
        self.filter_rule_action.triggered.connect(self.on_filter_rules)
        self.user_manual_action.triggered.connect(self.on_user_manual)
        self.about_action.triggered.connect(self.on_about)

        self.import_btn.clicked.connect(self.on_import)
        self.refresh_btn.clicked.connect(self.on_refresh)
        self.export_report_btn.clicked.connect(self.on_export_report)
        self.apply_date_btn.clicked.connect(self.on_apply_date)
        self.date_range_combo.currentTextChanged.connect(self.on_date_range_changed)

        self.detail_table.cellDoubleClicked.connect(self.on_cell_double_clicked)

    # ------------------ 业务逻辑（与之前相同，略作保留）------------------
    def _refresh_data(self):
        start = self.start_date_edit.date().toString("yyyy-MM-dd")
        end = self.end_date_edit.date().toString("yyyy-MM-dd")
        self.current_df = self.db.load_by_date_range(start, end)
        self.record_count_label.setText(f"记录数: {self.db.get_record_count()}")
        self.last_import_label.setText(f"上次导入: {getattr(self, 'last_import_time', '未导入')}")
        self.status_label.setText(f"当前显示 {len(self.current_df)} 条记录 ({start} ~ {end})")

    def _populate_detail_table(self):
        if self.current_df.empty:
            self.detail_table.setRowCount(0)
            return

        self.detail_table.setRowCount(len(self.current_df))
        for i, row in self.current_df.iterrows():
            id_item = QTableWidgetItem(str(row.get('id', '')))
            self.detail_table.setItem(i, 0, id_item)
            time_str = row['transaction_time'].strftime('%Y-%m-%d %H:%M:%S') if pd.notna(row['transaction_time']) else ''
            self.detail_table.setItem(i, 1, QTableWidgetItem(time_str))
            self.detail_table.setItem(i, 2, QTableWidgetItem(str(row['counterparty'])))
            self.detail_table.setItem(i, 3, QTableWidgetItem(str(row['description'])))
            amount_text = f"{row['amount']:.2f}"
            self.detail_table.setItem(i, 4, QTableWidgetItem(amount_text))
            type_text = '收入' if row['type'] == 'income' else '支出'
            self.detail_table.setItem(i, 5, QTableWidgetItem(type_text))
            self.detail_table.setItem(i, 6, QTableWidgetItem(str(row.get('payment_method', ''))))
            self.detail_table.setItem(i, 7, QTableWidgetItem(str(row.get('status', ''))))
            self.detail_table.setItem(i, 8, QTableWidgetItem(str(row.get('category', '未分类'))))

    @Slot()
    def on_import(self):
        file_paths, _ = QFileDialog.getOpenFileNames(
            self, "选择账单文件", "", "CSV文件 (*.csv);;Excel文件 (*.xlsx *.xls);;所有文件 (*)"
        )
        if not file_paths:
            return

        self.status_label.setText("正在导入并清洗数据...")
        QApplication.processEvents()

        try:
            df_list = []
            for path in file_paths:
                df = load_and_clean(path, custom_filter_keywords=self.custom_filter_keywords)
                if not df.empty:
                    df_list.append(df)

            if not df_list:
                QMessageBox.information(self, "导入结果", "未发现有效交易记录。")
                self.status_label.setText("导入完成，但无有效记录")
                return

            combined = merge_and_deduplicate(df_list)
            combined['category'] = combined.apply(
                lambda row: self.category_manager.classify(row['description'], row['counterparty']),
                axis=1
            )
            new_count = self.db.insert_transactions(combined)

            self.last_import_time = QDate.currentDate().toString("yyyy-MM-dd")
            self._refresh_data()
            self.on_refresh()
            self.status_label.setText(f"成功导入 {new_count} 条新记录")

        except Exception as e:
            QMessageBox.critical(self, "导入失败", f"处理文件时发生错误：\n{str(e)}")
            self.status_label.setText("导入失败")

    @Slot()
    def on_export_data(self):
        if self.current_df.empty:
            QMessageBox.warning(self, "导出数据", "没有数据可导出，请先导入账单。")
            return
        path, _ = QFileDialog.getSaveFileName(self, "导出数据", "", "Excel文件 (*.xlsx);;CSV文件 (*.csv)")
        if path:
            try:
                if path.endswith('.xlsx'):
                    self.current_df.to_excel(path, index=False, engine='openpyxl')
                else:
                    self.current_df.to_csv(path, index=False, encoding='utf-8-sig')
                QMessageBox.information(self, "导出成功", f"数据已导出至：{path}")
            except Exception as e:
                QMessageBox.critical(self, "导出失败", str(e))

    @Slot()
    def on_category_rules(self):
        dialog = CategoryRuleDialog(self.category_manager, self)
        dialog.rules_changed.connect(self.reclassify_all)
        dialog.exec()

    def reclassify_all(self):
        progress = QProgressDialog("正在重新分类所有记录...", "取消", 0, 0, self)
        progress.setWindowModality(Qt.WindowModal)
        progress.show()
        QApplication.processEvents()

        conn = sqlite3.connect(self.db.db_path)
        cursor = conn.cursor()
        cursor.execute("SELECT id, description, counterparty FROM transactions WHERE user_modified = 0")
        rows = cursor.fetchall()
        total = len(rows)
        progress.setMaximum(total)

        for i, (tid, desc, cp) in enumerate(rows):
            if progress.wasCanceled():
                break
            new_cat = self.category_manager.classify(desc or '', cp or '')
            if new_cat != '未分类':
                cursor.execute("UPDATE transactions SET category = ? WHERE id = ?", (new_cat, tid))
            if i % 100 == 0:
                progress.setValue(i)
                QApplication.processEvents()
        conn.commit()
        conn.close()
        progress.setValue(total)

        self._refresh_data()
        self.on_refresh()

    @Slot()
    def on_filter_rules(self):
        QMessageBox.information(self, "过滤规则", "过滤规则设置功能开发中。")

    @Slot()
    def on_user_manual(self):
        QMessageBox.information(self, "使用手册", "使用手册待完善。")

    @Slot()
    def on_about(self):
        QMessageBox.about(self, "关于账本助手",
                          "账本助手 v1.0\n\n"
                          "一款微信/支付宝账单自动化记录与可视化分析工具。\n"
                          "采用 Python + PySide6 开发，数据本地存储。\n"
                          "作者：Gavin\n"
                          "本软件仅用于个人财务管理。")

    @Slot()
    def on_refresh(self):
        if self.current_df.empty:
            self.status_label.setText("当前无数据，请先导入账单。")
            self.detail_table.setRowCount(0)
            for canvas in [self.overview_canvas, self.category_canvas, self.calendar_canvas,
                           self.income_canvas, self.high_freq_canvas]:
                canvas.figure.clear()
                canvas.draw()
            return

        start = self.start_date_edit.date().toString("yyyy-MM-dd")
        end = self.end_date_edit.date().toString("yyyy-MM-dd")
        self.status_label.setText(f"正在刷新报表（{start} 至 {end}）...")
        QApplication.processEvents()

        self._populate_detail_table()

        from report_generator import (
            plot_income_expense_trend, plot_category_pie,
            plot_calendar_heatmap, plot_income_sources, plot_top_counterparties
        )

        fig1 = plot_income_expense_trend(self.current_df, start, end)
        self.overview_canvas.figure = fig1
        self.overview_canvas.draw()

        fig2 = plot_category_pie(self.current_df)
        self.category_canvas.figure = fig2
        self.category_canvas.draw()

        start_date = QDate.fromString(start, "yyyy-MM-dd")
        year = start_date.year()
        month = start_date.month()
        fig3 = plot_calendar_heatmap(self.current_df, year, month)
        self.calendar_canvas.figure = fig3
        self.calendar_canvas.draw()

        fig4 = plot_income_sources(self.current_df)
        self.income_canvas.figure = fig4
        self.income_canvas.draw()

        fig5 = plot_top_counterparties(self.current_df, top_n=10)
        self.high_freq_canvas.figure = fig5
        self.high_freq_canvas.draw()

        self.status_label.setText(f"报表已刷新（{start} 至 {end}）")

    @Slot()
    def on_apply_date(self):
        start = self.start_date_edit.date().toString("yyyy-MM-dd")
        end = self.end_date_edit.date().toString("yyyy-MM-dd")
        self.status_label.setText(f"应用日期范围: {start} 至 {end}")
        self._refresh_data()
        self.on_refresh()

    @Slot(str)
    def on_date_range_changed(self, text):
        if text == "自定义":
            self.start_date_edit.setEnabled(True)
            self.end_date_edit.setEnabled(True)
            self.apply_date_btn.setEnabled(True)
        else:
            self.start_date_edit.setEnabled(False)
            self.end_date_edit.setEnabled(False)
            self.apply_date_btn.setEnabled(False)
            today = QDate.currentDate()
            if text == "本月":
                start = QDate(today.year(), today.month(), 1)
                end = today
            elif text == "上月":
                first_day_this_month = QDate(today.year(), today.month(), 1)
                last_day_last_month = first_day_this_month.addDays(-1)
                start = QDate(last_day_last_month.year(), last_day_last_month.month(), 1)
                end = last_day_last_month
            elif text == "本年度":
                start = QDate(today.year(), 1, 1)
                end = today
            else:
                return
            self.start_date_edit.setDate(start)
            self.end_date_edit.setDate(end)
            self._refresh_data()
            self.on_refresh()

    @Slot(int, int)
    def on_cell_double_clicked(self, row, column):
        if column == 8:
            id_item = self.detail_table.item(row, 0)
            if not id_item:
                return
            transaction_id = int(id_item.text())
            current_category = self.detail_table.item(row, column).text()
            categories = ["餐饮", "交通", "购物", "娱乐", "住房", "收入", "转账", "其他", "未分类"]
            category, ok = QInputDialog.getItem(self, "修改分类", "选择新分类:", categories, 0, False)
            if ok and category:
                self.db.update_category(transaction_id, category)
                self.detail_table.setItem(row, column, QTableWidgetItem(category))
                idx = self.current_df[self.current_df['id'] == transaction_id].index
                if len(idx) > 0:
                    self.current_df.loc[idx[0], 'category'] = category

    @Slot()
    def on_export_report(self):
        if self.current_df.empty:
            QMessageBox.warning(self, "导出报告", "没有数据可导出，请先导入账单。")
            return

        export_type, ok = QInputDialog.getItem(
            self, "导出报告", "请选择导出格式:",
            ["PDF 文件 (多页)", "图片文件 (每个图表单独保存)"], 0, False
        )
        if not ok:
            return

        figures_with_titles = [
            (self.overview_canvas.figure, "收支趋势"),
            (self.category_canvas.figure, "分类支出占比"),
            (self.calendar_canvas.figure, "消费热力图"),
            (self.income_canvas.figure, "收入来源"),
            (self.high_freq_canvas.figure, "高频交易对方")
        ]
        valid_figures = [(fig, title) for fig, title in figures_with_titles if fig.get_axes()]

        if not valid_figures:
            QMessageBox.warning(self, "导出报告", "没有有效的图表可导出。")
            return

        if export_type == "PDF 文件 (多页)":
            start_date = self.start_date_edit.date().toString("yyyy-MM-dd")
            end_date = self.end_date_edit.date().toString("yyyy-MM-dd")
            default_name = f"账本助手_{start_date}_to_{end_date}.pdf"
            filepath, _ = QFileDialog.getSaveFileName(
                self, "保存 PDF 报告", default_name, "PDF 文件 (*.pdf)"
            )
            if filepath:
                try:
                    export_to_pdf(valid_figures, filepath)
                    QMessageBox.information(self, "导出成功", f"报告已导出至：{filepath}")
                except Exception as e:
                    QMessageBox.critical(self, "导出失败", str(e))
        else:
            dir_path = QFileDialog.getExistingDirectory(self, "选择保存图片的文件夹")
            if dir_path:
                try:
                    start = self.start_date_edit.date().toString("yyyyMMdd")
                    end = self.end_date_edit.date().toString("yyyyMMdd")
                    images = [(fig, f"{title}_{start}_{end}") for fig, title in valid_figures]
                    saved = export_to_images(images, dir_path, dpi=150)
                    QMessageBox.information(self, "导出成功", f"已保存 {len(saved)} 个图片文件到：{dir_path}")
                except Exception as e:
                    QMessageBox.critical(self, "导出失败", str(e))


if __name__ == "__main__":
    app = QApplication(sys.argv)
    app.setApplicationName("账本助手")
    window = MainWindow()
    window.show()
    sys.exit(app.exec())
```

## 四、 非功能需求保证

性能：通过SQLite索引优化查询，pandas进行高效运算。在处理1万条记录时，整个流程（导入+清洗+报表生成）预估在10-15秒内完成，远优于30秒的要求。

稳定性：所有文件操作和数据入库操作都将包裹在try...except块中，任何单次导入的失败都不会影响已有数据。使用logging模块记录所有错误和关键信息。

安全性：代码中不包含任何网络请求（除检查更新外，该功能可选，且不会上传任何数据）。数据库文件不加密，但完全由用户本地保管。

可维护性：代码严格遵循PEP 8规范，各功能模块（导入、清洗、分类、绘图）分文件存放，并提供详细的docstring。提供requirements.txt。

## 五、可能出现的报错

文件读失败: sequence item 1: expected str instance, float found

我在弹出这个报错时是因为盲目升级了pandas等库的版本跳得太猛，导致与旧代码不兼容了

### 🚀 第一步：立刻“冻结”当前环境（备份）

这是最重要的一步，就像给现在的电脑状态拍个快照。在项目文件夹的地址栏输入 `cmd` 并回车，执行以下命令：

```bash
pip freeze > requirements.txt
```

这个命令会把当前所有包的版本信息都记录下来，存到 `requirements.txt` 文件里。万一操作失误，可以靠它快速还原。

### 🗺️ 第二步：检查“罪魁祸首”的版本

这个错误很可能是 **pandas 3.0** 版本引起的，它更新了一些核心机制，导致旧代码不兼容。为了确认情况，可以查看一下关键包的版本：

```bash
pip show pandas openpyxl PySide6
```

如果你的 `pandas`​ 版本是 ​**3.0.0 或更高**，那它就是问题根源。

### 🔧 第三步：将依赖库回滚到兼容版本

回滚的核心就是​**卸载新版本，然后安装一个确定能用的旧版本**。可以执行下面的命令，为几个关键库安装一个稳定的旧版本：

```bash
pip uninstall pandas openpyxl PySide6 matplotlib seaborn -y

pip install pandas==2.2.3 openpyxl==3.1.2 PySide6==6.7.3 matplotlib==3.9.2 seaborn==0.13.2
```

‍
