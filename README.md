# T3RN.BOT
T3RN多钱包同时运行 很好用
#!/bin/bash

# 检查是否为 root 用户
if [ "$EUID" -ne 0 ]; then
    echo -e "请使用 sudo 运行此脚本"
    exit 1
fi

# 定义仓库地址和目录名称
REPO_URL="https://github.com/sdohuajia/t3rn-bot.git"
DIR_NAME="t3rn-bot"
PYTHON_FILE="keys_and_addresses.py"
BOT_FILE="bot.py"
VENV_DIR="t3rn-env"  # 虚拟环境目录

# 检查是否安装了 git
if ! command -v git &> /dev/null; then
    echo "Git 未安装，请先安装 Git。"
    exit 1
fi

# 检查是否安装了 python3-pip 和 python3-venv
if ! command -v pip3 &> /dev/null; then
    echo "pip 未安装，正在安装 python3-pip..."
    sudo apt update
    sudo apt install -y python3-pip
fi

if ! python3 -m venv &> /dev/null; then
    echo "python3-venv 未安装，正在安装 python3-venv..."
    sudo apt update
    sudo apt install -y python3-venv
fi

# 拉取仓库
if [ -d "$DIR_NAME" ]; then
    echo "目录 $DIR_NAME 已存在，拉取最新更新..."
    cd "$DIR_NAME" || exit
    git pull origin main
else
    echo "正在克隆仓库 $REPO_URL..."
    git clone "$REPO_URL"
    cd "$DIR_NAME" || exit
fi

echo "已进入目录 $DIR_NAME"

# 创建虚拟环境并激活
echo "正在创建虚拟环境..."
python3 -m venv "$VENV_DIR"
source "$VENV_DIR/bin/activate"

# 升级 pip
echo "正在升级 pip..."
pip install --upgrade pip

# 安装依赖
echo "正在安装依赖 web3 和 colorama..."
pip install web3 colorama

# 提醒用户私钥安全
echo "警告：请务必确保您的私钥安全！"
echo "私钥应当保存在安全的位置，切勿公开分享或泄漏给他人。"

# 让用户输入私钥、标签和 Gas 值
echo "请输入您的私钥（多个私钥以空格分隔）："
read -r private_keys_input

echo "请输入您的标签（多个标签以空格分隔，与私钥顺序一致）："
read -r labels_input

echo "请输入每个钱包的 ARB-OP 的 Gas 值（单位: wei，以空格分隔，与私钥顺序一致）："
read -r arb_op_gas_values_input

echo "请输入每个钱包的 OP-ARB 的 Gas 值（单位: wei，以空格分隔，与私钥顺序一致）："
read -r op_arb_gas_values_input

# 检查输入是否一致
IFS=' ' read -r -a private_keys <<< "$private_keys_input"
IFS=' ' read -r -a labels <<< "$labels_input"
IFS=' ' read -r -a arb_op_gas_values <<< "$arb_op_gas_values_input"
IFS=' ' read -r -a op_arb_gas_values <<< "$op_arb_gas_values_input"

if [ "${#private_keys[@]}" -ne "${#labels[@]}" ] || \
   [ "${#private_keys[@]}" -ne "${#arb_op_gas_values[@]}" ] || \
   [ "${#private_keys[@]}" -ne "${#op_arb_gas_values[@]}" ]; then
    echo "私钥、标签、ARB-OP Gas 值和 OP-ARB Gas 值数量不一致，请重新运行脚本并确保它们匹配！"
    exit 1
fi

# 写入 keys_and_addresses.py 文件
echo "正在写入 $PYTHON_FILE 文件..."
cat > $PYTHON_FILE <<EOL
# 此文件由脚本生成

private_keys = [
$(printf "    '%s',\n" "${private_keys[@]}")
]

labels = [
$(printf "    '%s',\n" "${labels[@]}")
]

arb_op_gas_values = [
$(printf "    '%s',\n" "${arb_op_gas_values[@]}")
]

op_arb_gas_values = [
$(printf "    '%s',\n" "${op_arb_gas_values[@]}")
]
EOL

echo "$PYTHON_FILE 文件已生成。"

# 在指定链上执行操作
execute_on_chain() {
    local chain="$1"
    local rpc_url="$2"
    local private_key="$3"
    local label="$4"
    local gas_value="$5"

    # 使用 web3 计算发送方地址
    local sender_address
    sender_address=$(python3 -c "
from web3 import Web3
private_key = '$private_key'
web3 = Web3()
sender_address = web3.eth.account.from_key(private_key).address
print(sender_address)
")

    echo "$label 在 $chain 链上进行跨链操作，发送方和接收方地址均为: $sender_address"
    echo "使用手动设置的 GAS 值: $gas_value wei"

    # 调用 Python 脚本执行操作
    python3 $BOT_FILE --chain "$chain" --private-key "$private_key" --label "$label" \
        --gas-price "$gas_value" --gas-limit 21000 --to-address "$sender_address"
}

# 并行运行每个钱包任务
for i in "${!private_keys[@]}"; do
    echo "开始处理钱包: ${labels[$i]} (${private_keys[$i]})"
    while true; do
        # 在 Arbitrum 上跨链到 Optimism
        execute_on_chain "arb" "https://arb1.arbitrum.io/rpc" "${private_keys[$i]}" "${labels[$i]}" "${arb_op_gas_values[$i]}"
        sleep $((RANDOM % 16 + 25))  # 随机延迟 25-40 秒
        # 在 Optimism 上跨链回到 Arbitrum
        execute_on_chain "op" "https://mainnet.optimism.io/" "${private_keys[$i]}" "${labels[$i]}" "${op_arb_gas_values[$i]}"
        sleep $((RANDOM % 16 + 25))  # 随机延迟 25-40 秒
    done &
done

# 等待所有任务完成
wait
echo "所有钱包任务已完成！"
