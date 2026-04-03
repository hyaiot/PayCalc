<template>
  <div class="container">
    <h1>劳务报酬计算器</h1>
    
    <div class="calculator">
      <div class="labor-tax-calc">
        <h2>劳务报酬预扣预缴个税计算</h2>
        
        <div class="tax-mode-toggle">
          <button 
            :class="['mode-button', { active: laborTax.mode === 'income' }]"
            @click="laborTax.mode = 'income'; calculateLaborTax()"
          >
            输入劳务报酬
          </button>
          <button 
            :class="['mode-button', { active: laborTax.mode === 'afterTax' }]"
            @click="laborTax.mode = 'afterTax'; calculateLaborTaxReverse()"
          >
            输入税后收入
          </button>
        </div>

        <div class="form-group">
          <label>{{ laborTax.mode === 'income' ? '劳务报酬（元）：' : '税后收入（元）：' }}</label>
          <input 
            v-if="laborTax.mode === 'income'"
            type="number" 
            v-model.number="laborTax.income" 
            min="0" 
            step="0.01"
            @input="handleInput(laborTax, 'income')"
          >
          <input 
            v-else
            type="number" 
            v-model.number="laborTax.afterTaxIncome" 
            min="0" 
            step="0.01"
            @input="handleInput(laborTax, 'afterTaxIncome')"
          >
        </div>

        <div class="result">
          <h3>计算结果</h3>
          <div class="result-item">
            <span>减除费用：</span>
            <span class="value">¥ {{ laborTax.deduction.toFixed(2) }} 元</span>
          </div>
          <div class="result-item">
            <span>应纳税所得额：</span>
            <span class="value">¥ {{ laborTax.taxableIncome.toFixed(2) }} 元</span>
          </div>
          <div class="result-item">
            <span>适用税率：</span>
            <span class="value">{{ laborTax.taxRate }}%</span>
          </div>
          <div class="result-item">
            <span>速算扣除数：</span>
            <span class="value">¥ {{ laborTax.quickDeduction.toFixed(2) }} 元</span>
          </div>
          <div class="result-item highlight">
            <span>应缴税款：</span>
            <span class="value">¥ {{ laborTax.taxAmount.toFixed(2) }}</span>
          </div>
          <div class="result-item highlight">
            <span>{{ laborTax.mode === 'income' ? '税后收入：' : '劳务报酬：' }}</span>
            <span class="value">¥ {{ laborTax.mode === 'income' ? laborTax.afterTaxIncome.toFixed(2) : laborTax.income.toFixed(2) }}</span>
          </div>
          <div class="button-container">
            <button class="calculate-btn" @click="handleCalculate">计算并记录</button>
          </div>
        </div>
      </div>
    </div>

    <div class="history" v-if="history.length > 0">
      <h3>计算历史</h3>
      <ul>
        <li v-for="(item, index) in history" :key="index">
          <span class="history-type">{{ getHistoryType(item.type) }}</span>
          <span class="history-detail">{{ item.detail }}</span>
          <span class="history-result">{{ item.result }}</span>
        </li>
      </ul>
      <button class="clear-history" @click="clearHistory">清空历史</button>
    </div>

    <div class="footer">
      <p class="author">作者：hyaiot</p>
    </div>
  </div>
</template>

<script setup>
import { reactive, ref } from 'vue'

// 劳务报酬个税计算相关数据
const laborTax = reactive({
  mode: 'income',          // 计算模式：'income'（正算）或 'afterTax'（反算）
  income: 0,               // 劳务报酬收入
  afterTaxIncome: 0,       // 税后收入
  deduction: 0,            // 减除费用
  taxableIncome: 0,        // 应纳税所得额
  taxRate: 0,              // 适用税率
  quickDeduction: 0,       // 速算扣除数
  taxAmount: 0             // 应缴税款
})

// 计算历史记录
const history = ref([])

// 计算劳务报酬个税（正算：从劳务报酬到税后收入）
const calculateLaborTax = () => {
  // 确定减除费用和应纳税所得额
  if (laborTax.income <= 800) {
    // 劳务报酬 ≤ 800元，不缴税
    laborTax.deduction = 0
    laborTax.taxableIncome = 0
    laborTax.taxRate = 0
    laborTax.quickDeduction = 0
    laborTax.taxAmount = 0
    laborTax.afterTaxIncome = laborTax.income
  } else if (laborTax.income <= 4000) {
    // 800 < 劳务报酬 ≤ 4000，减除费用800
    laborTax.deduction = 800
    laborTax.taxableIncome = laborTax.income - 800
  } else {
    // 劳务报酬 > 4000，减除费用20%
    laborTax.deduction = laborTax.income * 0.2
    laborTax.taxableIncome = laborTax.income * 0.8
  }
  
  // 税率表（三级累进）
  const laborTaxBrackets = [
    { min: 0, max: 20000, rate: 20, deduction: 0 },    // 0-20000元，税率20%
    { min: 20000, max: 50000, rate: 30, deduction: 2000 },  // 20000-50000元，税率30%，速算扣除数2000
    { min: 50000, max: Infinity, rate: 40, deduction: 7000 } // 50000元以上，税率40%，速算扣除数7000
  ]
  
  // 找到适用的税率档位
  const bracket = laborTaxBrackets.find(b => laborTax.taxableIncome > b.min && laborTax.taxableIncome <= b.max)
  
  if (bracket) {
    // 应用税率和速算扣除数
    laborTax.taxRate = bracket.rate
    laborTax.quickDeduction = bracket.deduction
    laborTax.taxAmount = laborTax.taxableIncome * (bracket.rate / 100) - bracket.deduction
  } else {
    // 默认情况（通常不会触发）
    laborTax.taxRate = 0
    laborTax.quickDeduction = 0
    laborTax.taxAmount = 0
  }
  
  // 确保税额不为负数
  laborTax.taxAmount = Math.max(0, parseFloat(laborTax.taxAmount.toFixed(6)))
  // 计算税后收入
  laborTax.afterTaxIncome = parseFloat((laborTax.income - laborTax.taxAmount).toFixed(6))
  
  // 最终结果保留两位小数
  laborTax.taxAmount = parseFloat(laborTax.taxAmount.toFixed(2))
  laborTax.afterTaxIncome = parseFloat(laborTax.afterTaxIncome.toFixed(2))
  

}

// 计算劳务报酬个税（反算：从税后收入到劳务报酬）
const calculateLaborTaxReverse = () => {
  // 定义不同的减除费用场景
  const scenarios = [
    // 场景1：劳务报酬 ≤ 800元，不缴税
    {
      name: 'no tax',
      condition: (income) => income <= 800,  // 条件：劳务报酬 ≤ 800
      calculate: (afterTax) => {
        return {
          income: afterTax,          // 劳务报酬 = 税后收入
          deduction: 0,              // 减除费用 = 0
          taxableIncome: 0,          // 应纳税所得额 = 0
          taxRate: 0,                // 税率 = 0
          quickDeduction: 0,         // 速算扣除数 = 0
          taxAmount: 0               // 应缴税款 = 0
        }
      }
    },
    // 场景2：800 < 劳务报酬 ≤ 4000，减除费用800
    {
      name: 'deduction 800',
      condition: (income) => income > 800 && income <= 4000,  // 条件：800 < 劳务报酬 ≤ 4000
      calculate: (afterTax) => {
        // 公式：(税后收入 - 160) / 0.8 = 劳务报酬
        let income = (afterTax - 160) / 0.8  // 计算劳务报酬
        
        // 尝试调整精度，确保反算结果与正算一致
        for (let adjust = -0.02; adjust <= 0.02; adjust += 0.01) {
          const adjustedIncome = parseFloat((income + adjust).toFixed(2))
          
          // 验证是否符合场景条件
          if (adjustedIncome > 800 && adjustedIncome <= 4000) {
            // 计算正算结果，验证是否与输入的税后收入一致
            const adjustedTaxableIncome = adjustedIncome - 800
            const adjustedTax = adjustedTaxableIncome * 0.2
            const calculatedAfterTax = parseFloat((adjustedIncome - adjustedTax).toFixed(2))
            const targetAfterTax = parseFloat(afterTax.toFixed(2))
            
            if (calculatedAfterTax === targetAfterTax) {
              const taxRate = 20                     // 税率固定为20%
              const quickDeduction = 0               // 速算扣除数为0
              const taxAmount = parseFloat((adjustedTaxableIncome * (taxRate / 100) - quickDeduction).toFixed(6))  // 计算应缴税款：应纳税所得额 × 税率 - 速算扣除数
              
              // 最终结果保留两位小数
              const finalTaxAmount = parseFloat(taxAmount.toFixed(2))
              
              return {
                income: adjustedIncome,          // 劳务报酬
                deduction: 800,   // 减除费用
                taxableIncome: adjustedTaxableIncome,    // 应纳税所得额
                taxRate,          // 税率
                quickDeduction,   // 速算扣除数
                taxAmount: finalTaxAmount         // 应缴税款
              }
            }
          }
        }
        
        // 如果没有找到精确匹配，使用原始计算
        const taxableIncome = income - 800     // 计算应纳税所得额
        const taxRate = 20                     // 税率固定为20%
        const quickDeduction = 0               // 速算扣除数为0
        const taxAmount = taxableIncome * (taxRate / 100) - quickDeduction  // 计算应缴税款
        
        return {
          income,          // 劳务报酬
          deduction: 800,   // 减除费用
          taxableIncome,    // 应纳税所得额
          taxRate,          // 税率
          quickDeduction,   // 速算扣除数
          taxAmount         // 应缴税款
        }
      }
    },
    // 场景3：劳务报酬 > 4000，减除费用20%
    {
      name: 'deduction 20%',
      condition: (income) => income > 4000,  // 条件：劳务报酬 > 4000
      calculate: (afterTax) => {
        // 税率表（三级累进）
        const laborTaxBrackets = [
          { min: 0, max: 20000, rate: 20, deduction: 0 },    // 0-20000元，税率20%
          { min: 20000, max: 50000, rate: 30, deduction: 2000 },  // 20000-50000元，税率30%，速算扣除数2000
          { min: 50000, max: Infinity, rate: 40, deduction: 7000 } // 50000元以上，税率40%，速算扣除数7000
        ]
        
        // 从高到低尝试不同税率档位
        for (let i = laborTaxBrackets.length - 1; i >= 0; i--) {
          const bracket = laborTaxBrackets[i]
          const { rate, deduction: quickDeduction } = bracket
          
          // 直接尝试可能的劳务报酬值，从反推值附近开始
          const baseIncome = (afterTax - quickDeduction) / (1 - (rate / 100) * 0.8)
          const startIncome = Math.floor(baseIncome * 100) / 100 - 0.10
          
          // 收集所有可能的匹配值
          const matchingIncomes = []
          
          // 尝试从startIncome到startIncome+0.20的所有可能值（步长0.01）
          for (let income = startIncome; income <= startIncome + 0.20; income += 0.01) {
            const adjustedIncome = parseFloat(income.toFixed(2))
            const adjustedTaxableIncome = parseFloat((adjustedIncome * 0.8).toFixed(6))
            
            // 验证是否符合当前税率级别
            if (adjustedTaxableIncome > bracket.min && adjustedTaxableIncome <= bracket.max) {
              // 计算正算结果，验证是否与输入的税后收入一致
              const adjustedTax = parseFloat((adjustedTaxableIncome * (rate / 100) - quickDeduction).toFixed(6))
              const calculatedAfterTax = parseFloat((adjustedIncome - adjustedTax).toFixed(2))
              const targetAfterTax = parseFloat(afterTax.toFixed(2))
              
              if (calculatedAfterTax === targetAfterTax) {
                matchingIncomes.push({
                  income: adjustedIncome,
                  taxableIncome: adjustedTaxableIncome,
                  tax: adjustedTax
                })
              }
            }
          }
          
          // 如果找到多个匹配值，选择最大的那个（更接近用户期望的结果）
          if (matchingIncomes.length > 0) {
            // 按劳务报酬降序排序
            matchingIncomes.sort((a, b) => b.income - a.income)
            
            const bestMatch = matchingIncomes[0]
            const deduction = parseFloat((bestMatch.income * 0.2).toFixed(6))
            const taxAmount = parseFloat(bestMatch.tax.toFixed(6))
            
            // 最终结果保留两位小数
            const finalDeduction = parseFloat(deduction.toFixed(2))
            const finalTaxAmount = parseFloat(taxAmount.toFixed(2))
            
            return {
              income: bestMatch.income,          // 劳务报酬
              deduction: finalDeduction,       // 减除费用
              taxableIncome: bestMatch.taxableIncome,    // 应纳税所得额
              taxRate: rate,    // 税率
              quickDeduction,   // 速算扣除数
              taxAmount: finalTaxAmount         // 应缴税款
            }
          }
        }
        
        // 如果没有找到精确匹配，使用原始计算
        for (let i = laborTaxBrackets.length - 1; i >= 0; i--) {
          const bracket = laborTaxBrackets[i]
          const { rate, deduction: quickDeduction } = bracket
          
          const income = (afterTax - quickDeduction) / (1 - (rate / 100) * 0.8)
          const taxableIncome = income * 0.8
          
          if (taxableIncome > bracket.min && taxableIncome <= bracket.max) {
            const deduction = income * 0.2
            const taxAmount = income - afterTax
            
            return {
              income,
              deduction,
              taxableIncome,
              taxRate: rate,
              quickDeduction,
              taxAmount
            }
          }
        }
        
        // 默认情况（通常不会触发）
        return {
          income: afterTax,
          deduction: 0,
          taxableIncome: 0,
          taxRate: 0,
          quickDeduction: 0,
          taxAmount: 0
        }
      }
    }
  ]
  
  // 尝试所有场景，找到最合适的
  let bestResult = null
  
  for (const scenario of scenarios) {
    const result = scenario.calculate(laborTax.afterTaxIncome)  // 计算每种场景的结果
    if (scenario.condition(result.income)) {  // 验证结果是否符合场景条件
      bestResult = result  // 找到合适的场景
      break
    }
  }
  
  if (bestResult) {
    // 应用计算结果并修复浮点数精度
    laborTax.income = parseFloat(bestResult.income.toFixed(2))
    laborTax.deduction = parseFloat(bestResult.deduction.toFixed(2))
    laborTax.taxableIncome = parseFloat(bestResult.taxableIncome.toFixed(2))
    laborTax.taxRate = bestResult.taxRate
    laborTax.quickDeduction = bestResult.quickDeduction
    laborTax.taxAmount = parseFloat(bestResult.taxAmount.toFixed(2))
  } else {
    // 默认情况（通常不会触发）
    laborTax.taxRate = 0
    laborTax.quickDeduction = 0
    laborTax.taxableIncome = 0
    laborTax.income = parseFloat(laborTax.afterTaxIncome.toFixed(2))
    laborTax.taxAmount = 0
    laborTax.deduction = 0
  }
  

}

// 添加计算记录到历史
const addToHistory = (type, detail, result) => {
  const timestamp = new Date().toLocaleTimeString()  // 获取当前时间
  history.value.unshift({ type, detail, result, timestamp })  // 添加到历史记录开头
  if (history.value.length > 10) {
    history.value.pop()  // 保持历史记录最多10条
  }
}

// 清空历史记录
const clearHistory = () => {
  history.value = []
}

// 获取历史记录类型的中文名称
const getHistoryType = (type) => {
  return type === 'laborTax' ? '劳务' : type
}

// 处理输入，限制两位小数
const handleInput = (obj, key) => {
  if (obj[key] !== null && obj[key] !== undefined) {
    // 限制两位小数
    obj[key] = parseFloat(obj[key].toFixed(2))
  }
}

// 处理计算按钮点击
const handleCalculate = () => {
  // 确保输入值为两位小数
  if (laborTax.income !== null && laborTax.income !== undefined) {
    laborTax.income = parseFloat(laborTax.income.toFixed(2))
  }
  if (laborTax.afterTaxIncome !== null && laborTax.afterTaxIncome !== undefined) {
    laborTax.afterTaxIncome = parseFloat(laborTax.afterTaxIncome.toFixed(2))
  }
  
  // 根据当前模式执行计算
  if (laborTax.mode === 'income') {
    calculateLaborTax()
    // 修复浮点数精度问题
    laborTax.afterTaxIncome = parseFloat(laborTax.afterTaxIncome.toFixed(2))
    // 添加到历史记录
    if (laborTax.income > 0) {
      addToHistory('laborTax', `劳务¥${laborTax.income}`, `税后¥${laborTax.afterTaxIncome.toFixed(2)}`)
    }
  } else {
    calculateLaborTaxReverse()
    // 修复浮点数精度问题
    laborTax.income = parseFloat(laborTax.income.toFixed(2))
    // 添加到历史记录
    if (laborTax.afterTaxIncome > 0) {
      addToHistory('laborTax', `税后¥${laborTax.afterTaxIncome}`, `劳务¥${laborTax.income.toFixed(2)}`)
    }
  }
}

// 初始化计算（首次加载时执行）
calculateLaborTax()
</script>

<style scoped>
.container {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Microsoft YaHei', sans-serif;
}

.button-container {
  margin-top: 20px;
  text-align: center;
}

.calculate-btn {
  background-color: #4CAF50;
  color: white;
  border: none;
  padding: 10px 20px;
  font-size: 16px;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.calculate-btn:hover {
  background-color: #45a049;
}

h1 {
  text-align: center;
  color: #333;
  margin-bottom: 30px;
}

.calculator {
  background: #f9f9f9;
  padding: 25px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

h2 {
  color: #555;
  margin-bottom: 20px;
  font-size: 18px;
}

.tax-mode-toggle {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.mode-button {
  flex: 1;
  padding: 10px;
  border: 2px solid #ddd;
  background: white;
  cursor: pointer;
  font-size: 14px;
  border-radius: 6px;
  transition: all 0.3s;
}

.mode-button:hover {
  border-color: #4CAF50;
}

.mode-button.active {
  background: #4CAF50;
  color: white;
  border-color: #4CAF50;
}

.form-group {
  margin-bottom: 20px;
}

.form-group label {
  display: block;
  margin-bottom: 8px;
  color: #666;
  font-weight: 500;
}

.form-group input {
  width: 100%;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 16px;
  box-sizing: border-box;
  transition: border-color 0.3s;
}

.form-group input:focus {
  outline: none;
  border-color: #4CAF50;
}

.result {
  margin-top: 25px;
  padding: 20px;
  background: white;
  border-radius: 8px;
  border-left: 4px solid #4CAF50;
}

.result h3 {
  margin: 0 0 15px 0;
  color: #333;
  font-size: 16px;
}

.result-item {
  display: flex;
  justify-content: space-between;
  padding: 10px 0;
  border-bottom: 1px solid #eee;
  color: #666;
}

.result-item:last-child {
  border-bottom: none;
}

.result-item.highlight {
  font-size: 20px;
  font-weight: bold;
  color: #4CAF50;
  margin-top: 10px;
  padding-top: 15px;
  border-top: 2px solid #4CAF50;
}

.result-item .value {
  font-weight: 600;
}

.history {
  margin-top: 30px;
  padding: 20px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.history h3 {
  margin: 0 0 15px 0;
  color: #333;
}

.history ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.history li {
  display: flex;
  justify-content: space-between;
  padding: 10px;
  border-bottom: 1px solid #eee;
  font-size: 14px;
}

.history li:last-child {
  border-bottom: none;
}

.history-type {
  font-weight: bold;
  color: #4CAF50;
  min-width: 60px;
}

.history-detail {
  flex: 1;
  color: #666;
  text-align: center;
}

.history-result {
  font-weight: bold;
  color: #333;
  min-width: 100px;
  text-align: right;
}

.clear-history {
  margin-top: 15px;
  padding: 8px 16px;
  background: #f44336;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: background 0.3s;
  display: block;
  margin-left: auto;
  margin-right: auto;
}

.clear-history:hover {
  background: #d32f2f;
}

.footer {
  margin-top: 30px;
  text-align: center;
  padding: 15px;
}

.author {
  color: #999;
  font-size: 14px;
  margin: 0;
}
</style>
