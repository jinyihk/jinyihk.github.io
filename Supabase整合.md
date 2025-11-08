# 一步步将 Supabase 整合到您的网页中

我来详细解释如何将 Supabase 整合到您的网页，并解决您关于云端部署的问题。

## 1. npm 是什么？

**npm (Node Package Manager)** 是 JavaScript 的包管理器，类似于：
- 手机应用商店（但用于代码库）
- 让您可以轻松安装和管理代码依赖项
- 几乎所有现代 JavaScript 项目都使用它

## 2. 云端部署解决方案

您完全不需要一直打开电脑！以下是推荐的云端部署方案：

### 推荐平台：
- **Vercel** (最适合静态网站)
- **Netlify** (功能强大，免费额度高)
- **GitHub Pages** (完全免费)

这些平台都可以：
- 自动从您的代码仓库部署
- 当您更新代码时自动重新部署
- 完全在云端运行，无需您的电脑开机

## 3. 完整整合步骤

### 步骤 1: 创建 Supabase 项目
1. 访问 [supabase.com](https://supabase.com)
2. 点击 "Start your project"
3. 使用 GitHub 或邮箱注册
4. 创建新项目：
   - 项目名称：`golden-ease-advisor`
   - 数据库密码：设置强密码
   - 地区：选择 **亚太地区-新加坡** (离香港最近)
5. 等待项目初始化完成（约2分钟）

### 步骤 2: 获取 API 密钥
项目创建后：
1. 进入 **Settings > API**
2. 记录下：
   - **URL** (如：`https://xxxxx.supabase.co`)
   - **anon public key** (以 `eyJ` 开头)

### 步骤 3: 设置数据库表

在 Supabase 左侧菜单进入 **SQL Editor**，执行以下 SQL：

```sql
-- 创建课程表
CREATE TABLE courses (
  id SERIAL PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  image_url TEXT,
  instructor VARCHAR(100),
  duration VARCHAR(50),
  price DECIMAL(10,2),
  category VARCHAR(50),
  created_at TIMESTAMP DEFAULT NOW()
);

-- 创建用户资料表（扩展认证用户）
CREATE TABLE profiles (
  id UUID REFERENCES auth.users(id) PRIMARY KEY,
  username VARCHAR(100),
  company_name VARCHAR(255),
  industry VARCHAR(100),
  score INTEGER DEFAULT 0,
  membership_type VARCHAR(50) DEFAULT 'basic',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- 启用行级安全
ALTER TABLE courses ENABLE ROW LEVEL SECURITY;
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

-- 创建策略：所有人都可以读取课程
CREATE POLICY "任何人都可以查看课程" ON courses
  FOR SELECT USING (true);

-- 创建策略：用户只能查看和修改自己的资料
CREATE POLICY "用户可查看自己的资料" ON profiles
  FOR SELECT USING (auth.uid() = id);

CREATE POLICY "用户可更新自己的资料" ON profiles
  FOR UPDATE USING (auth.uid() = id);
```

### 步骤 4: 修改您的网页代码

在您的 HTML 文件中添加 Supabase 客户端：

```html
<!-- 在 </body> 标签前添加 -->
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<script>
  // 初始化 Supabase 客户端 - 替换为您的实际密钥
  const supabaseUrl = 'https://您的项目.supabase.co'
  const supabaseKey = '您的anon公钥'
  const supabase = supabase.createClient(supabaseUrl, supabaseKey)
  
  // 会员功能
  async function signUp(email, password, userData) {
    const { data, error } = await supabase.auth.signUp({
      email: email,
      password: password,
    })
    
    if (data.user) {
      // 创建用户资料
      const { error } = await supabase
        .from('profiles')
        .insert([
          { 
            id: data.user.id, 
            username: userData.username,
            company_name: userData.companyName,
            industry: userData.industry
          }
        ])
      
      if (!error) {
        alert('注册成功！请检查邮箱验证。')
      }
    }
  }
  
  async function signIn(email, password) {
    const { data, error } = await supabase.auth.signInWithPassword({
      email: email,
      password: password,
    })
    
    if (data.user) {
      alert('登录成功！')
      // 更新界面显示登录状态
      updateLoginStatus(data.user)
    }
  }
  
  async function updateUserScore(points) {
    const { data: { user } } = await supabase.auth.getUser()
    if (user) {
      const { error } = await supabase
        .from('profiles')
        .update({ score: points })
        .eq('id', user.id)
    }
  }
  
  // 课程功能
  async function loadCourses() {
    let { data: courses, error } = await supabase
      .from('courses')
      .select('*')
    
    if (!error) {
      displayCourses(courses)
    }
  }
  
  async function searchCourses(keyword) {
    let { data: courses, error } = await supabase
      .from('courses')
      .select('*')
      .or(`title.ilike.%${keyword}%,description.ilike.%${keyword}%`)
    
    if (!error) {
      displayCourses(courses)
    }
  }
  
  function displayCourses(courses) {
    const container = document.getElementById('courses-container')
    container.innerHTML = ''
    
    courses.forEach(course => {
      const courseCard = `
        <div class="course-card">
          <div class="course-img" style="background-image: url('${course.image_url}')"></div>
          <div class="course-content">
            <h3 class="course-title">${course.title}</h3>
            <p class="course-description">${course.description}</p>
            <div class="course-meta">
              <span>${course.instructor}</span>
              <span>${course.duration}</span>
            </div>
            <div class="course-price">HK$ ${course.price}</div>
            <button class="enroll-btn">立即报名</button>
          </div>
        </div>
      `
      container.innerHTML += courseCard
    })
  }
  
  // 页面加载时获取课程
  document.addEventListener('DOMContentLoaded', function() {
    loadCourses()
    
    // 搜索功能
    document.querySelector('.search-btn').addEventListener('click', function() {
      const keyword = document.querySelector('.search-box input').value
      searchCourses(keyword)
    })
  })
</script>
```

### 步骤 5: 添加测试数据

在 Supabase 的 **Table Editor** 中，为 courses 表添加一些测试数据：

```sql
INSERT INTO courses (title, description, image_url, instructor, duration, price, category) VALUES
('大湾区营商法律实务与风险防范', '深入解析大湾区营商法律环境，掌握关键法律风险防范策略，保障企业稳健发展。', 'https://images.unsplash.com/photo-1552664730-d307ca884978', '王律师', '2小时', 1288.00, '法律合规'),
('数字营销策略：从零到千万营收', '学习高效的数字营销策略，掌握社交媒体、内容营销和SEO等关键技能，实现业务快速增长。', 'https://images.unsplash.com/photo-1460925895917-afdab827c52f', '张总监', '3小时', 1288.00, '营销推广');
```

### 步骤 6: 部署到云端

1. **将代码上传到 GitHub**
   - 在 GitHub 创建新仓库
   - 上传您的 HTML、CSS、JS 文件

2. **使用 Vercel 部署**
   - 访问 [vercel.com](https://vercel.com)
   - 使用 GitHub 登录
   - 导入您的仓库
   - Vercel 会自动部署，并提供永久可访问的网址

3. **配置环境变量**
   - 在 Vercel 项目设置中，添加环境变量：
     - `SUPABASE_URL`: 您的 Supabase URL
     - `SUPABASE_ANON_KEY`: 您的 Supabase anon key

## 4. 完整的功能实现

### 会员系统功能：
- ✅ 会员注册/登录
- ✅ 密码修改（通过邮件重置）
- ✅ 会员资料存储
- ✅ 会员积分系统

### 课程系统功能：
- ✅ 课程数据从数据库加载
- ✅ 课程搜索和筛选
- ✅ 动态课程卡片生成

## 5. 费用和限制总结

**Supabase 免费套餐**：
- 数据库：500MB
- 认证用户：50,000个
- API 请求：每月50,000次
- 文件存储：1GB
- 完全足够初期使用

**Vercel 免费套餐**：
- 无限网站部署
- 每月100GB带宽
- 自动SSL证书
- 完全免费用于个人项目

这样您的整个系统就完全在云端运行了，无需您的电脑开机，用户可以通过您获得的 Vercel 网址随时访问！