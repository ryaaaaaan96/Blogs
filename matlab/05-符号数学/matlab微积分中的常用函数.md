
# MATLAB符号数学与微积分

## 📋 概述

MATLAB的符号数学工具箱（Symbolic Math Toolbox）提供了强大的符号计算功能，可以进行精确的数学运算，包括微积分、代数、微分方程求解等。本文将详细介绍MATLAB中符号数学的使用方法和微积分运算。

## 🔧 符号变量创建

### 基本符号运算

```matlab
% 1. 创建符号变量的方法
x = sym('x');                    % 创建单个符号变量
syms x y z;                      % 创建多个符号变量
syms t real;                     % 创建实数符号变量
syms a positive;                 % 创建正数符号变量

% 2. 符号表达式
f = x^2 + 2*x + 1;              % 创建符号表达式
g = sin(x) + cos(x);            % 三角函数表达式
h = exp(x) + log(x);            % 指数和对数表达式
```

### 合并同类项

```matlab
% collect函数 - 合并同类项
syms x t;

% 基本合并
EXPR = sym('(x^2+x*exp(-t)+1)*(x+exp(-t))');
expr1 = collect(EXPR);                      % 按x合并
expr2 = collect(EXPR, 'exp(-t)');          % 按exp(-t)合并

% 展开后合并
expanded = expand((x+1)^3 + 2*x^2);
collected = collect(expanded);

fprintf('原表达式: %s\n', char(EXPR));
fprintf('按x合并: %s\n', char(expr1));
fprintf('按exp(-t)合并: %s\n', char(expr2));
```

**注解：**
1. **这里会报出警告，但不影响程序的运行**
2. **exp(-t)的意思就是e的-t次方**

## 🔀 因式分解

```matlab
% factor函数 - 符号计算的因式分解
syms x a;

% 1. 除了x外不含其他自由变量的情况
f1 = x^4 - 5*x^3 + 5*x^2 + 5*x - 6;
factor_f1 = factor(f1);
fprintf('f1分解: %s\n', char(factor_f1));

% 2. 含其他自由变量的情况
f2 = x^2 - a^2;
factor_f2 = factor(f2);
fprintf('f2分解: %s\n', char(factor_f2));

% 3. 对正整数的质数分解
factor_1025 = factor(1025);
fprintf('1025的质因数分解: %s\n', char(factor_1025));

% 更多因式分解示例
syms x y;
f3 = x^3 - y^3;                % 立方差
f4 = x^4 - 1;                  % 四次方减1
f5 = x^6 - y^6;                % 六次方差

factor_f3 = factor(f3);
factor_f4 = factor(f4);  
factor_f5 = factor(f5);

fprintf('x^3-y^3分解: %s\n', char(factor_f3));
fprintf('x^4-1分解: %s\n', char(factor_f4));
fprintf('x^6-y^6分解: %s\n', char(factor_f5));
```

## 📈 展开

```matlab
% expand函数 - 对符号多项式或函数进行展开
syms x y;

% 多项式展开
poly1 = (x + 1)^3;
expanded_poly1 = expand(poly1);
fprintf('(x+1)^3展开: %s\n', char(expanded_poly1));

poly2 = (x + y)^4;
expanded_poly2 = expand(poly2);
fprintf('(x+y)^4展开: %s\n', char(expanded_poly2));

% 三角函数展开
trig1 = sin(x + y);
expanded_trig1 = expand(trig1);
fprintf('sin(x+y)展开: %s\n', char(expanded_trig1));

trig2 = cos(2*x);
expanded_trig2 = expand(trig2);
fprintf('cos(2x)展开: %s\n', char(expanded_trig2));

% 指数函数展开
exp1 = exp(x + y);
expanded_exp1 = expand(exp1);
fprintf('exp(x+y)展开: %s\n', char(expanded_exp1));

% 对数函数展开
log1 = log(x*y);
expanded_log1 = expand(log1);
fprintf('log(xy)展开: %s\n', char(expanded_log1));
```

## 🧮 符号表达式化简

```matlab
% simplify和simple函数 - 对符号表达式进行化简
syms x;

% 复杂表达式示例
f = (1/x^3 + 6/x^2 + 12/x + 8)^(1/3);

% 1. 使用simplify简化
sfy1 = simplify(f);
sfy2 = simplify(sfy1);
fprintf('simplify第一次: %s\n', char(sfy1));
fprintf('simplify第二次: %s\n', char(sfy2));

% 2. 其他化简函数
g = (x^2 - 1)/(x - 1);
simplified_g = simplify(g);
fprintf('(x^2-1)/(x-1)化简: %s\n', char(simplified_g));

% 三角函数化简
h = sin(x)^2 + cos(x)^2;
simplified_h = simplify(h);
fprintf('sin^2(x)+cos^2(x)化简: %s\n', char(simplified_h));

% 指数化简
k = exp(x)*exp(y);
simplified_k = simplify(k);
fprintf('exp(x)*exp(y)化简: %s\n', char(simplified_k));

% 对数化简
m = log(x) + log(y);
simplified_m = simplify(m);
fprintf('log(x)+log(y)化简: %s\n', char(simplified_m));
```

**说明：**
- `simplify`函数对表达式进行化简
- 在较新版本的MATLAB中，`simple`函数已被弃用，建议使用`simplify`
- `simplify`可以设置参数来控制化简过程

### 其他化简函数

```matlab
syms x y;

% collect - 合并同类项
expr1 = x^2 + 2*x + x^2 + 3*x;
collected_expr = collect(expr1);
fprintf('collect: %s -> %s\n', char(expr1), char(collected_expr));

% expand - 展开表达式
expr2 = (x + 1)*(x - 1);
expanded_expr = expand(expr2);
fprintf('expand: %s -> %s\n', char(expr2), char(expanded_expr));

% factor - 因式分解
expr3 = x^2 - 1;
factored_expr = factor(expr3);
fprintf('factor: %s -> %s\n', char(expr3), char(factored_expr));

% combine - 合并表达式
expr4 = sin(x)*cos(x);
combined_expr = combine(expr4, 'sincos');
fprintf('combine: %s -> %s\n', char(expr4), char(combined_expr));

% rewrite - 重写表达式
expr5 = tan(x);
rewritten_expr = rewrite(expr5, 'sincos');
fprintf('rewrite: %s -> %s\n', char(expr5), char(rewritten_expr));
```

## 📉 极限运算

```matlab
% limit函数 - 计算极限
syms x;

% 基本极限计算
% limit(f, x, a) 计算f在x趋于a时的极限
% limit(f, x, a, 'left') 左极限
% limit(f, x, a, 'right') 右极限

% 例题1：计算基本极限
y1 = (1 + 4*x)^(1/x);
y2 = (exp(x) - 1)/x;

limit_y1 = limit(y1, x, 0);
limit_y2 = limit(y2, x, 0);

fprintf('lim(x->0)(1+4x)^(1/x) = %s\n', char(limit_y1));
fprintf('lim(x->0)(e^x-1)/x = %s\n', char(limit_y2));

% 左右极限
y3 = sqrt(x) - 2^(-1/x);
right_limit = limit(y3, x, 0, 'right');
left_limit = limit(y3, x, 0, 'left');

fprintf('右极限: %s\n', char(right_limit));
if ~isempty(left_limit)
    fprintf('左极限: %s\n', char(left_limit));
else
    fprintf('左极限不存在\n');
end

% 无穷大处的极限
y4 = (x^2 + 1)/(x^2 - 1);
inf_limit = limit(y4, x, inf);
fprintf('lim(x->∞)(x^2+1)/(x^2-1) = %s\n', char(inf_limit));

% 更多极限示例
limits_examples = {
    sin(x)/x, 0, 'sin(x)/x在x->0';
    (1 + 1/x)^x, inf, '(1+1/x)^x在x->∞';
    (1 - cos(x))/x^2, 0, '(1-cos(x))/x^2在x->0';
    x*sin(1/x), inf, 'x*sin(1/x)在x->∞'
};

for i = 1:size(limits_examples, 1)
    func = limits_examples{i, 1};
    point = limits_examples{i, 2};
    description = limits_examples{i, 3};
    
    lim_result = limit(func, x, point);
    fprintf('%s = %s\n', description, char(lim_result));
end
```

**注意：在左右极限不相等或左右极限有一个不存在时，MATLAB的默认状态是求右极限。**

## 📊 求导

### 一元函数求导

```matlab
% diff函数 - 求导
% diff(f) - 对默认变量求一阶导数
% diff(f, n) - 求n阶导数
% diff(f, x) - 对变量x求导
% diff(f, x, n) - 对变量x求n阶导数

syms x;

% 基本求导示例
f = 3*x^3 + 5*x + 1;
df_dx = diff(f);                % 一阶导数
d2f_dx2 = diff(f, 2);           % 二阶导数

fprintf('f(x) = %s\n', char(f));
fprintf('f''(x) = %s\n', char(df_dx));
fprintf('f''''(x) = %s\n', char(d2f_dx2));

% 更多函数的求导
functions = {
    sin(x), 'sin(x)';
    exp(x), 'exp(x)';
    log(x), 'log(x)';
    x^x, 'x^x';
    sqrt(x), 'sqrt(x)';
    1/x, '1/x';
    tan(x), 'tan(x)';
    asin(x), 'asin(x)'
};

fprintf('\n求导示例:\n');
for i = 1:size(functions, 1)
    func = functions{i, 1};
    name = functions{i, 2};
    derivative = diff(func);
    fprintf('d/dx[%s] = %s\n', name, char(derivative));
end

% 链式法则示例
composite_funcs = {
    sin(x^2), 'sin(x^2)';
    exp(cos(x)), 'exp(cos(x))';
    log(sin(x)), 'log(sin(x))';
    (x^2 + 1)^(1/2), 'sqrt(x^2+1)'
};

fprintf('\n复合函数求导:\n');
for i = 1:size(composite_funcs, 1)
    func = composite_funcs{i, 1};
    name = composite_funcs{i, 2};
    derivative = diff(func);
    fprintf('d/dx[%s] = %s\n', name, char(derivative));
end
```

### 求导数值计算

```matlab
% 在特定点计算导数值
syms x;
y = 3*x^3 - 2*x + 1;
dy_dx = diff(y);

% 在x=1处计算导数值
x_val = 1;
derivative_at_1 = subs(dy_dx, x, x_val);
% 或者使用eval (但subs更安全)
% derivative_at_1 = eval(subs(dy_dx, x, 1));

fprintf('f(x) = %s\n', char(y));
fprintf('f''(x) = %s\n', char(dy_dx));
fprintf('f''(1) = %s\n', char(derivative_at_1));

% 数值求导示例
x_values = 0:0.1:2;
y_values = 3*x_values.^3 - 2*x_values + 1;
dy_values = 9*x_values.^2 - 2;   % 解析导数

% 绘制函数和导数
figure;
subplot(2,1,1);
plot(x_values, y_values, 'b-', 'LineWidth', 2);
title('原函数 f(x) = 3x^3 - 2x + 1');
xlabel('x');
ylabel('f(x)');
grid on;

subplot(2,1,2);
plot(x_values, dy_values, 'r-', 'LineWidth', 2);
title('导函数 f''(x) = 9x^2 - 2');
xlabel('x');
ylabel('f''(x)');
grid on;
```

## 🔄 多元函数的偏导数

```matlab
% 多元函数偏导数
% diff(f, xi) - 对变量xi求偏导
% diff(f, xi, n) - 对变量xi求n阶偏导

syms x y z;

% 二元函数示例
f_2d = x^2*sin(2*y);
df_dx = diff(f_2d, x);          % 对x的偏导数
df_dy = diff(f_2d, y);          % 对y的偏导数

fprintf('f(x,y) = %s\n', char(f_2d));
fprintf('∂f/∂x = %s\n', char(df_dx));
fprintf('∂f/∂y = %s\n', char(df_dy));

% 高阶偏导数
d2f_dx2 = diff(f_2d, x, 2);     % 二阶偏导数 ∂²f/∂x²
d2f_dy2 = diff(f_2d, y, 2);     % 二阶偏导数 ∂²f/∂y²
d2f_dxdy = diff(diff(f_2d, x), y); % 混合偏导数 ∂²f/∂x∂y

fprintf('∂²f/∂x² = %s\n', char(d2f_dx2));
fprintf('∂²f/∂y² = %s\n', char(d2f_dy2));
fprintf('∂²f/∂x∂y = %s\n', char(d2f_dxdy));

% 三元函数示例
f_3d = x^2*y + y^2*z + z^2*x;
gradient_f = [diff(f_3d, x), diff(f_3d, y), diff(f_3d, z)];

fprintf('\nf(x,y,z) = %s\n', char(f_3d));
fprintf('梯度 ∇f = [%s, %s, %s]\n', char(gradient_f(1)), char(gradient_f(2)), char(gradient_f(3)));

% 计算散度和旋度示例
syms x y z;
F = [x*y, y*z, z*x];           % 向量场

% 散度 div F = ∂Fx/∂x + ∂Fy/∂y + ∂Fz/∂z
div_F = diff(F(1), x) + diff(F(2), y) + diff(F(3), z);

% 旋度的z分量 (curl F)_z = ∂Fy/∂x - ∂Fx/∂y
curl_F_z = diff(F(2), x) - diff(F(1), y);

fprintf('向量场 F = [%s, %s, %s]\n', char(F(1)), char(F(2)), char(F(3)));
fprintf('散度 div F = %s\n', char(div_F));
fprintf('旋度z分量 (curl F)_z = %s\n', char(curl_F_z));
```

## ∫ 积分运算

### 一元函数的不定积分

```matlab
% int函数 - 积分运算
% int(f) - 对默认变量的不定积分
% int(f, v) - 对变量v的不定积分
% int(f, a, b) - 定积分
% int(f, v, a, b) - 对变量v从a到b的定积分

syms x z;

% 基本不定积分
basic_integrals = {
    x^2, 'x^2';
    sin(x), 'sin(x)';
    exp(x), 'exp(x)';
    1/x, '1/x';
    1/(1+x^2), '1/(1+x^2)';
    sqrt(x), 'sqrt(x)'
};

fprintf('基本不定积分:\n');
for i = 1:size(basic_integrals, 1)
    func = basic_integrals{i, 1};
    name = basic_integrals{i, 2};
    integral_result = int(func);
    fprintf('∫%s dx = %s\n', name, char(integral_result));
end

% 复杂积分示例
y1 = 1/(sin(x)^2*cos(x)^2);
int_y1 = int(y1);
fprintf('\n复杂积分:\n');
fprintf('∫1/(sin²x·cos²x) dx = %s\n', char(int_y1));

% 显示美化结果
fprintf('美化显示:\n');
pretty(int_y1);

% 对不同变量积分
f_multi = x/(1 + z^2);
int_f_z = int(f_multi, z);     % 对z积分
int_f_x = int(f_multi, x);     % 对x积分

fprintf('∫x/(1+z²) dz = %s\n', char(int_f_z));
fprintf('∫x/(1+z²) dx = %s\n', char(int_f_x));
```

### 一元函数的定积分

```matlab
% 定积分计算
syms x;

% 基本定积分示例
definite_integrals = {
    x^2, -1, 1, 'x^2从-1到1';
    sin(x), 0, pi, 'sin(x)从0到π';
    exp(x), 0, 1, 'exp(x)从0到1';
    1/(1+x^2), -inf, inf, '1/(1+x^2)从-∞到∞'
};

fprintf('定积分计算:\n');
for i = 1:size(definite_integrals, 1)
    func = definite_integrals{i, 1};
    a = definite_integrals{i, 2};
    b = definite_integrals{i, 3};
    description = definite_integrals{i, 4};
    
    integral_result = int(func, x, a, b);
    fprintf('∫%s = %s\n', description, char(integral_result));
end

% 具体数值计算
y1 = x^2 + sin(x)/(1 + x^2);
int_y1 = int(y1, x, -1, 1);
double_result = double(int_y1);  % 转换为数值

fprintf('\n∫_{-1}^{1} [x² + sin(x)/(1+x²)] dx = %s ≈ %.6f\n', char(int_y1), double_result);

% 比较解析解和数值解
y2 = x^2/(1 + x^2);
analytic = int(y2, x, -1, 1);
numeric = double(analytic);

fprintf('∫_{-1}^{1} x²/(1+x²) dx = %s ≈ %.6f\n', char(analytic), numeric);
```

### 多重积分运算

```matlab
% 二重积分计算
syms x y;

% 计算不定积分 ∫∫f(x,y) dy dx
f_2d = x^2 + y^2 + 1;

% 先对y积分，再对x积分
inner_integral = int(f_2d, y);          % ∫f dy
outer_integral = int(inner_integral, x); % ∫(∫f dy) dx

fprintf('f(x,y) = %s\n', char(f_2d));
fprintf('∫f dy = %s\n', char(inner_integral));
fprintf('∫∫f dy dx = %s\n', char(outer_integral));

% 计算定积分 ∫_{0}^{1} ∫_{x}^{x+1} (x²+y²+1) dy dx
% 注意：内层积分的上下限可以是x的函数
definite_2d = int(int(f_2d, y, x, x+1), x, 0, 1);
fprintf('∫_{0}^{1} ∫_{x}^{x+1} (x²+y²+1) dy dx = %s\n', char(definite_2d));

% 更复杂的积分区域
% 计算 ∫_{0}^{1} ∫_{0}^{√(1-x²)} xy dy dx (第一象限单位圆)
f_circle = x*y;
circle_integral = int(int(f_circle, y, 0, sqrt(1-x^2)), x, 0, 1);
fprintf('单位圆第一象限内 ∫∫xy dy dx = %s\n', char(circle_integral));

% 三重积分示例
syms z;
f_3d = x*y*z;

% ∫∫∫ xyz dz dy dx 在单位立方体内
triple_integral = int(int(int(f_3d, z, 0, 1), y, 0, 1), x, 0, 1);
fprintf('单位立方体内 ∫∫∫xyz dz dy dx = %s\n', char(triple_integral));
```

## 📊 数值微分与数值积分

### 数值微分

```matlab
% 数值微分 - 用离散方法近似计算导数
% diff函数用于数值数据时计算差分

% 创建数据
x_data = 0:0.1:2*pi;
y_data = sin(x_data);

% 数值微分（前向差分）
dy_dx_numeric = diff(y_data) ./ diff(x_data);
x_diff = x_data(1:end-1);       % 差分后长度减1

% 解析导数用于比较
dy_dx_analytic = cos(x_data);

% 绘制比较
figure;
subplot(2,1,1);
plot(x_data, y_data, 'b-', 'LineWidth', 2);
title('原函数 y = sin(x)');
xlabel('x');
ylabel('y');
grid on;

subplot(2,1,2);
plot(x_diff, dy_dx_numeric, 'r-', 'LineWidth', 2);
hold on;
plot(x_data, dy_dx_analytic, 'b--', 'LineWidth', 2);
title('导数比较');
xlabel('x');
ylabel('dy/dx');
legend('数值导数', '解析导数', 'Location', 'best');
grid on;

% 计算误差
error = abs(dy_dx_numeric - cos(x_diff));
max_error = max(error);
fprintf('数值微分最大误差: %.6f\n', max_error);
```

### 数值积分函数

```matlab
% MATLAB数值积分函数

% 1. integral函数 - 一维数值积分
func1 = @(x) x.^2;
result1 = integral(func1, 0, 1);
fprintf('∫_{0}^{1} x² dx = %.6f (数值)\n', result1);
fprintf('解析解 = %.6f\n', 1/3);

% 2. integral2函数 - 二维数值积分
func2 = @(x,y) x.*y;
result2 = integral2(func2, 0, 1, 0, 1);
fprintf('∫_{0}^{1} ∫_{0}^{1} xy dy dx = %.6f (数值)\n', result2);
fprintf('解析解 = %.6f\n', 0.25);

% 3. 复杂函数的数值积分
% 高斯函数积分
gaussian = @(x) exp(-x.^2);
gauss_integral = integral(gaussian, -inf, inf);
gauss_analytic = sqrt(pi);
fprintf('∫_{-∞}^{∞} e^{-x²} dx = %.6f (数值)\n', gauss_integral);
fprintf('解析解 = %.6f\n', gauss_analytic);

% 振荡函数积分
oscillating = @(x) sin(x)./x;
osc_integral = integral(oscillating, 0.1, 10);
fprintf('∫_{0.1}^{10} sin(x)/x dx = %.6f\n', osc_integral);

% 4. 参数化积分
% ∫_{0}^{1} x^a dx, 其中a为参数
parametric_integral = @(a) integral(@(x) x.^a, 0, 1);
a_values = [0.5, 1, 1.5, 2, 2.5];
fprintf('\n参数化积分 ∫_{0}^{1} x^a dx:\n');
for a = a_values
    numerical = parametric_integral(a);
    analytical = 1/(a+1);
    fprintf('a = %.1f: 数值解 = %.6f, 解析解 = %.6f\n', a, numerical, analytical);
end
```

### 数值积分精度控制

```matlab
% 积分精度控制
func_precision = @(x) exp(sin(x));

% 不同精度设置
tolerances = [1e-3, 1e-6, 1e-9, 1e-12];
fprintf('\n积分精度测试 ∫_{0}^{π} e^{sin(x)} dx:\n');

for tol = tolerances
    tic;
    result = integral(func_precision, 0, pi, 'AbsTol', tol, 'RelTol', tol);
    time_elapsed = toc;
    fprintf('容差 = %.0e: 结果 = %.10f, 耗时 = %.4f秒\n', tol, result, time_elapsed);
end

% 比较不同积分方法的性能
functions_test = {
    @(x) sin(x), 0, pi, 2, 'sin(x)';
    @(x) x.^2, 0, 1, 1/3, 'x²';
    @(x) exp(x), 0, 1, exp(1)-1, 'e^x';
    @(x) 1./(1+x.^2), -1, 1, pi/2, '1/(1+x²)'
};

fprintf('\n数值积分精度验证:\n');
for i = 1:size(functions_test, 1)
    func = functions_test{i, 1};
    a = functions_test{i, 2};
    b = functions_test{i, 3};
    exact = functions_test{i, 4};
    name = functions_test{i, 5};
    
    numerical = integral(func, a, b);
    error = abs(numerical - exact);
    
    fprintf('∫%s: 数值=%.8f, 精确=%.8f, 误差=%.2e\n', name, numerical, exact, error);
end
```

## 🚀 高级应用示例

### 符号微分方程

```matlab
% 符号微分方程求解
syms y(x);

% 一阶线性ODE: dy/dx + y = x
ode1 = diff(y, x) + y == x;
sol1 = dsolve(ode1);
fprintf('dy/dx + y = x 的通解: %s\n', char(sol1));

% 带初始条件的ODE
ode2 = diff(y, x) + y == x;
sol2 = dsolve(ode2, y(0) == 1);
fprintf('dy/dx + y = x, y(0)=1 的特解: %s\n', char(sol2));

% 二阶ODE: d²y/dx² + y = 0
ode3 = diff(y, x, 2) + y == 0;
sol3 = dsolve(ode3);
fprintf('d²y/dx² + y = 0 的通解: %s\n', char(sol3));
```

### 级数展开

```matlab
% 泰勒级数展开
syms x;

% 常用函数的泰勒展开
functions_taylor = {
    exp(x), 'e^x';
    sin(x), 'sin(x)';
    cos(x), 'cos(x)';
    log(1+x), 'log(1+x)';
    1/(1-x), '1/(1-x)'
};

fprintf('泰勒级数展开（在x=0处，5阶）:\n');
for i = 1:size(functions_taylor, 1)
    func = functions_taylor{i, 1};
    name = functions_taylor{i, 2};
    taylor_series = taylor(func, x, 'Order', 6);
    fprintf('%s = %s + O(x^6)\n', name, char(taylor_series));
end

% 在非零点的展开
f = sin(x);
taylor_at_pi2 = taylor(f, x, pi/2, 'Order', 4);
fprintf('\nsin(x)在x=π/2处的泰勒展开: %s\n', char(taylor_at_pi2));
```

---

> **符号数学总结**：MATLAB的符号数学工具箱提供了强大的精确计算能力。掌握符号变量创建、表达式化简、微积分运算是进行理论分析的基础。结合数值计算方法，可以验证理论结果并处理复杂的实际问题。

1. **除了x外不含其他自由变量的情况**  
    sym a x; f1=x^4-5_x^3+5_x^2+5*x-6; factor(f1)  
    
2. **含其他自由变量的情况之一**  
    f2-x^2-a^2; factor(f2)  
    
3. **对正整数的质数分解**  
    factor(1025)  
    

  

## 展开

> expand(S) 对符号多项式或函数S进行展开

例如：

> syms x y; expand((x+1)^3) expand(sin(x+y))

  

## 对符号表达式进行化简

> r=simple(S) 或 r=simplify(S) 对符号表达式S进行化简

  

![](https://pic1.zhimg.com/80/v2-91dd813aefd552e73e67134ec584b694_720w.webp)

image.png

  

1. 运用simplify简化  
    syms x; f=(1/x^3+6/x^2+12/x+8)^(1/3); sfy1=simplify(f), sfy1=simplify(sfy1)  
    
2. 运用simple简化(更加简化一些)  
    g1=simple(f), g2=simple(g1)  
    

simplify和simple是[Matlab](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3DMatlab%26spm%3D1001.2101.3001.7020)符号数学工具箱提供的两个du简化函数，区别如下：  
simplify的调zhi用格式为：simplify(S)；对表达式S进行化简。  
simple是通过对dao表达式尝试多种不同的方法（包括simplify）进行化简，以寻求符号表达式S的最简形式。  
调用方式为：  
[r,how]=simple(S);r为返回的简化形式，how为化简过程中使用的一种方法。how有以下几种形式：  
（1）simplify 函数对表达式进行化简；  
（2）radsimp函数对含根式的表达式进行化简；  
（3）combine 函数将表达式中以求和、乘积、幂运算等形式出现的项进行合并；  
（4）collet合并同类项  
（5）factor函数实现因式分解  
（6)convert函数完成表达式形式的转换

simplify可以设置参数,这里可以直接help simplify来查看相关参数,simple函数似乎已经在新版本中被剔除

## 极限的运算

  

![](https://pic1.zhimg.com/80/v2-03db0f61c1afca9ffe006245a32235b4_720w.webp)

image.png

  

> **注意：在左右极限不相等或左右极限有一个不存在时，MATLAB的默认状态是求右极限。**

  

### 例题1

  

![](https://pic3.zhimg.com/80/v2-38252c194ef6f71f0de046fb5f5b59ba_720w.webp)

image.png

  

> syms x;  
> y1=(1+4*x)^(1/x);  
> y2=(exp(x)-1) /x;  
> limit(y1,x,0)  
> limit(y2,x,0)

  

![](https://pic3.zhimg.com/80/v2-0e0007155349d9749ac969d03173e762_720w.webp)

image.png

  

> syms x;  
> y=sqrt(x)-2^(-1/x);  
> limit(y,x,0, 'right')

  

## 求导

> **一元函数的求导** diff(f)  
> **对n求导** diff(f,n)

  

![](https://pic4.zhimg.com/80/v2-102a3befaba9443917c274018b98fcff_720w.webp)

image.png

  

> syms x;  
> f=3_x^3+5_x+1;  
> diff(f,2)

  

![](https://pic3.zhimg.com/80/v2-2dcb920338759c444ef8842012f235e2_720w.webp)

image.png

  

> syms x;  
> y=3_x^3-2_x+1;  
> B=diff(y),x=1;  
> eval(B)  
> ps:eval函数的功能是将字符串转换为matlab可执行语句。通俗而言，比如 你输入 a=’b=1’; 会在workspace里看见生成了变量a，a的类型是字符串，字符串的内容是b=1 然后你输入eval(a) 就会看见变量里生成了变量b，b是一个1乘1的double型矩阵，元素的值为1 也就是说，执行eval(a)相当于执行a的内容，相当于执行b=1

  

## 多元函数的偏导数

> diff(f,xi) diff(f,xi,n)

  

![](https://pic4.zhimg.com/80/v2-bf0df527b75e743a683de9ec1a220ed3_720w.webp)

image.png

  

> syms x y;  
> z=x^2_sin(2_y);  
> B=diff(z,x)

  

## 积分运算

  

### 一元函数的不定积分

> int(f) 求函数f对默认变量的不定积分,用于函数只有**一个变量**的情况 int(f,v) 求符号函数f对变量v的不定积分

例:  

![](https://pic4.zhimg.com/80/v2-df4652da941fa10cd726b408771c5deb_720w.webp)

image.png

  

> sysms x; y=1/(sin(x)^2*cos(x)^2); int(y) pretty(int(y))  
> 解释pretty:  
> [pretty数学公式美化](https://link.zhihu.com/?target=https%3A//blog.csdn.net/weixin_43964993/article/details/108027252)  
> syms x z; B=int(x/(1+z^2),z)

  

### 一元函数的定积分

> int(f,x,a,b) 用微积分基本公式计算定积分

  

![](https://pic2.zhimg.com/80/v2-e78407c08aa7c76ab452c6b347c11881_720w.webp)

image.png

  

![](https://pic1.zhimg.com/80/v2-fb5a594df63349080b7bad64a963b0e0_720w.webp)

image.png

  

> syms x; y=(x^2+sin(x)/(1+x^2)); int(y,x,-1,1)  
> syms x; y=x^2/(1+x^2); int(y,x,-1,1)

  

### 多重积分运算

> int(int(f,y),x) 计算不定积分

![](https://pic4.zhimg.com/80/v2-16b2723a79f2b814760b02aabc4c9723_720w.webp)

image.png

  
int(int(f,y,c,d),x,a,b) 计算不定积分

![](https://pic1.zhimg.com/80/v2-a90b5b19efc450280d7c85bd20583250_720w.webp)

image.png

  

例:  

![](https://pic3.zhimg.com/80/v2-70856d62634d5af573ad4bbcacce337e_720w.webp)

image.png

  

> syms x,y int(int((x^2+y^2+1),x,x+1),y,0,1)#错误代码  
> 这里注意y在里面,x在外面  
> int(int(x^2+y^2+1,y,x,x+1),x,0,1)

  

## 数值微分与数值积分

  

### 数值微分

> 数值微分是用离散的方法近似地计算函数y=f(x)在某点x=a处的导数值，通常仅当函数以离散数值形式给出时才有这种必要。  
> diff(x)

**一、MATLAB中的积分函数**

在MATLAB中，有许多内置的积分函数，可以用于求解各种积分问题。以下是MATLAB中常用的积分函数。

**1. quad函数**

quad函数是MATLAB中最常用的积分函数之一，可以用于求解一维积分问题。quad函数的语法格式如下：

I = quad(fun,a,b)

其中，fun是要求解的函数，a和b是积分区间的上下限。I是积分结果。

**2. quad2d函数**

quad2d函数是MATLAB中用于求解二维积分问题的函数，可以用于求解二元函数在给定区域上的积分。quad2d函数的语法格式如下：

I = quad2d(fun,xmin,xmax,ymin,ymax)

其中，fun是要求解的二元函数，xmin、xmax、ymin和ymax是积分区域的上下限。I是积分结果。

**3. integral函数**

integral函数是MATLAB中用于求解一维积分问题的函数，可以用于求解固定区间上的积分。integral函数的语法格式如下：

I = integral(fun,a,b)

其中，fun是要求解的函数，a和b是积分区间的上下限。I是积分结果。

**4. integral2函数**

integral2函数是MATLAB中用于求解二维积分问题的函数，可以用于求解二元函数在给定区域上的积分。integral2函数的语法格式如下：

I = integral2(fun,xmin,xmax,ymin,ymax)

其中，fun是要求解的二元函数，xmin、xmax、ymin和ymax是积分区域的上下限。I是积分结果。

![](https://pics6.baidu.com/feed/b7003af33a87e95014f6b385a25e454ff9f2b4ec.jpeg@f_auto?token=c7a2fef7b908221994dd628e291d993f)

**二、MATLAB中积分的求解方法**

在MATLAB中，可以使用以上提到的积分函数来求解各种积分问题。以下是MATLAB中积分的求解方法。

**1. 使用quad函数求解一维积分**

使用quad函数可以求解一维积分问题。例如，要求解函数f(x)=x^2在区间[0,1]上的积分，可以使用以下代码：

syms x

f = x^2;

I = quad(f,0,1)

运行以上代码，MATLAB会返回积分结果I=0.3333。

**2. 使用quad2d函数求解二维积分**

使用quad2d函数可以求解二维积分问题。例如，要求解函数f(x,y)=x*y在区域[0,1]×[0,1]上的积分，可以使用以下代码：

syms x y

f = x*y;

I = quad2d(f,0,1,0,1)

运行以上代码，MATLAB会返回积分结果I=0.2500。

**3. 使用integral函数求解一维积分**

使用integral函数可以求解一维积分问题。例如，要求解函数f(x)=x^2在区间[0,1]上的积分，可以使用以下代码：

syms x

f = x^2;

I = integral(f,0,1)

运行以上代码，MATLAB会返回积分结果I=0.3333。

**4. 使用integral2函数求解二维积分**

使用integral2函数可以求解二维积分问题。例如，要求解函数f(x,y)=x*y在区域[0,1]×[0,1]上的积分，可以使用以下代码：

syms x y

f = x*y;

I = integral2(f,0,1,0,1)

运行以上代码，MATLAB会返回积分结果I=0.2500。

![](https://pics7.baidu.com/feed/6f061d950a7b02089a767214d1bfe4df562cc87f.jpeg@f_auto?token=ab04f5adb644d0c9fa6313ee2b8ed029)

在MATLAB中，可以使用quad、quad2d、integral和integral2等函数来求解各种积分问题。使用这些函数可以快速、准确地求解积分问题，提高工作效率。在使用这些函数时，需要注意输入参数的格式和取值范围，以确保求解结果的准确性。