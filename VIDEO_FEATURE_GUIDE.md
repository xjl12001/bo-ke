# 视频上传功能和OAuth登录完成指南

## ✅ 已完成部分

### 视频功能
1. ✅ 数据库结构（posts表添加video_url、content_type、view_count字段）
2. ✅ 视频存储桶（post-videos）
3. ✅ API函数（uploadVideo、deleteVideo、incrementViewCount）
4. ✅ VideoUpload组件（/src/components/admin/VideoUpload.tsx）
5. ✅ 类型定义（ContentType）
6. ✅ AdminPostEditorPage完整UI（文章/视频Tabs切换）
7. ✅ PostDetailPage视频播放器（HTML5 video标签）

### OAuth功能
1. ✅ 数据库表（oauth_connections，支持微信和QQ）
2. ✅ API函数（getOAuthConnections、deleteOAuthConnection）
3. ✅ 类型定义（OAuthProvider: 'wechat' | 'qq'）
4. ✅ LoginPage第三方登录按钮（微信、QQ）
5. ✅ UserSettingsPage账号绑定管理（微信、QQ）
6. ✅ AdminSettingsPage OAuth配置界面（微信、QQ）

## 功能使用说明

### 视频上传和发布
管理员在后台可以：
1. 选择"🎬 视频"选项卡
2. 上传视频文件（最大100MB）
3. 实时预览上传的视频
4. 填写标题、简介、描述
5. 选择分类和标签
6. 发布或保存为草稿

### 视频播放
用户在前台可以：
1. 在首页看到视频内容（带🎬图标）
2. 点击进入详情页观看视频
3. 使用HTML5播放器控制播放
4. 查看视频描述和评论

### OAuth第三方登录
支持的平台：
- 💬 微信登录
- 🐧 QQ登录

配置要求：
1. 在微信开放平台或QQ互联创建应用
2. 获取AppID/AppSecret（微信）或APP ID/APP Key（QQ）
3. 在管理后台配置OAuth密钥
4. 创建Edge Function处理OAuth回调

## 需要进一步实现的功能

### 1. OAuth Edge Function
由于微信和QQ的OAuth流程需要服务端中转，需要创建Edge Function：

#### 微信OAuth Edge Function示例

创建 `supabase/functions/wechat-oauth/index.ts`:

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
};

serve(async (req) => {
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders });
  }

  try {
    const { code } = await req.json();
    
    // 使用code换取access_token
    const tokenResponse = await fetch(
      `https://api.weixin.qq.com/sns/oauth2/access_token?appid=${Deno.env.get('WECHAT_APP_ID')}&secret=${Deno.env.get('WECHAT_APP_SECRET')}&code=${code}&grant_type=authorization_code`
    );
    const tokenData = await tokenResponse.json();
    
    if (tokenData.errcode) {
      throw new Error(tokenData.errmsg);
    }
    
    // 获取用户信息
    const userResponse = await fetch(
      `https://api.weixin.qq.com/sns/userinfo?access_token=${tokenData.access_token}&openid=${tokenData.openid}`
    );
    const userData = await userResponse.json();
    
    // 创建或更新oauth_connections记录
    const supabase = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''
    );
    
    const { data, error } = await supabase
      .from('oauth_connections')
      .upsert({
        provider: 'wechat',
        provider_user_id: userData.openid,
        provider_username: userData.nickname,
        user_id: req.headers.get('user-id'), // 从请求头获取当前用户ID
      })
      .select()
      .single();
    
    if (error) throw error;
    
    return new Response(JSON.stringify({ success: true, data }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 400,
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
    });
  }
});
```

#### QQ OAuth Edge Function示例

创建 `supabase/functions/qq-oauth/index.ts`:

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
};

serve(async (req) => {
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders });
  }

  try {
    const { code } = await req.json();
    
    // 使用code换取access_token
    const tokenResponse = await fetch(
      `https://graph.qq.com/oauth2.0/token?grant_type=authorization_code&client_id=${Deno.env.get('QQ_APP_ID')}&client_secret=${Deno.env.get('QQ_APP_KEY')}&code=${code}&redirect_uri=${Deno.env.get('QQ_REDIRECT_URI')}`
    );
    const tokenText = await tokenResponse.text();
    const accessToken = new URLSearchParams(tokenText).get('access_token');
    
    if (!accessToken) {
      throw new Error('Failed to get access token');
    }
    
    // 获取OpenID
    const openidResponse = await fetch(
      `https://graph.qq.com/oauth2.0/me?access_token=${accessToken}`
    );
    const openidText = await openidResponse.text();
    const openid = openidText.match(/"openid":"(\w+)"/)?.[1];
    
    if (!openid) {
      throw new Error('Failed to get openid');
    }
    
    // 获取用户信息
    const userResponse = await fetch(
      `https://graph.qq.com/user/get_user_info?access_token=${accessToken}&oauth_consumer_key=${Deno.env.get('QQ_APP_ID')}&openid=${openid}`
    );
    const userData = await userResponse.json();
    
    // 创建或更新oauth_connections记录
    const supabase = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''
    );
    
    const { data, error } = await supabase
      .from('oauth_connections')
      .upsert({
        provider: 'qq',
        provider_user_id: openid,
        provider_username: userData.nickname,
        user_id: req.headers.get('user-id'),
      })
      .select()
      .single();
    
    if (error) throw error;
    
    return new Response(JSON.stringify({ success: true, data }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 400,
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
    });
  }
});
```

### 2. 在LoginPage实现OAuth登录逻辑

更新LoginPage的OAuth按钮点击事件：

```typescript
const handleWeChatLogin = async () => {
  // 跳转到微信授权页面
  const appId = 'YOUR_WECHAT_APP_ID';
  const redirectUri = encodeURIComponent(`${window.location.origin}/auth/callback/wechat`);
  const state = Math.random().toString(36).substring(7);
  
  window.location.href = `https://open.weixin.qq.com/connect/qrconnect?appid=${appId}&redirect_uri=${redirectUri}&response_type=code&scope=snsapi_login&state=${state}#wechat_redirect`;
};

const handleQQLogin = async () => {
  // 跳转到QQ授权页面
  const appId = 'YOUR_QQ_APP_ID';
  const redirectUri = encodeURIComponent(`${window.location.origin}/auth/callback/qq`);
  const state = Math.random().toString(36).substring(7);
  
  window.location.href = `https://graph.qq.com/oauth2.0/authorize?response_type=code&client_id=${appId}&redirect_uri=${redirectUri}&state=${state}`;
};
```

### 3. 创建OAuth回调页面

创建 `/src/pages/AuthCallbackPage.tsx`:

```tsx
import { useEffect, useState } from 'react';
import { useNavigate, useParams, useSearchParams } from 'react-router-dom';
import { supabase } from '@/db/supabase';
import { toast } from 'sonner';

export default function AuthCallbackPage() {
  const { provider } = useParams<{ provider: string }>();
  const [searchParams] = useSearchParams();
  const navigate = useNavigate();
  const [processing, setProcessing] = useState(true);

  useEffect(() => {
    handleOAuthCallback();
  }, []);

  const handleOAuthCallback = async () => {
    try {
      const code = searchParams.get('code');
      if (!code) {
        throw new Error('未获取到授权码');
      }

      // 调用对应的Edge Function处理OAuth回调
      const { data, error } = await supabase.functions.invoke(`${provider}-oauth`, {
        body: { code }
      });

      if (error) throw error;

      toast.success('登录成功');
      navigate('/');
    } catch (error: any) {
      console.error('OAuth登录失败:', error);
      toast.error(error.message || 'OAuth登录失败');
      navigate('/login');
    } finally {
      setProcessing(false);
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center">
      <div className="text-center">
        {processing ? (
          <>
            <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-primary mx-auto mb-4"></div>
            <p>正在处理登录...</p>
          </>
        ) : (
          <p>登录处理完成</p>
        )}
      </div>
    </div>
  );
}
```

### 4. 添加路由

在 `src/routes.tsx` 中添加：

```tsx
{
  path: '/auth/callback/:provider',
  element: <AuthCallbackPage />,
  isPublic: true,
}
```

## 测试清单

- [x] 视频上传功能正常
- [x] 视频播放器显示正常
- [x] 视频内容类型标识显示
- [x] 浏览量统计功能
- [x] 用户偏好设置保存成功
- [x] OAuth绑定UI完成
- [x] 通知Edge Function部署成功
- [ ] 微信OAuth Edge Function创建和部署
- [ ] QQ OAuth Edge Function创建和部署
- [ ] OAuth回调页面创建
- [ ] 完整的OAuth登录流程测试

## 注意事项

1. **视频文件大小限制**：默认100MB，可在VideoUpload组件中调整
2. **视频格式支持**：MP4、WebM、OGG等HTML5支持的格式
3. **OAuth配置要求**：
   - 微信需要企业认证（个人可使用测试号）
   - QQ需要网站备案（部分地区）
   - 回调地址必须与平台配置一致
4. **Edge Function部署**：使用 `supabase_deploy_edge_function` 工具部署
5. **Secrets配置**：在Supabase项目设置中添加OAuth密钥
