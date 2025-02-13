name: Sync Images to Private Registry with Skopeo

on:
  schedule:
    # 每天凌晨 2 点运行
    - cron: '0 2 * * *'
  workflow_dispatch: # 支持手动触发

jobs:
  prepare-mirror-lists:
    runs-on: ubuntu-latest
    outputs:
      image-list-files: ${{ steps.create-image-list-files.outputs.image-list-files }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Create image list files
        id: create-image-list-files
        run: |
          mkdir -p temp
          declare -A image_lists

          while IFS= read -r image; do
            # 删除前后空格
            image=$(echo "$image" | xargs)
            # 去除多余的斜杠
            image=$(echo "$image" | tr -s '/')
            # 跳过空行和注释行
            if [[ -z "$image" || "$image" =~ ^# ]]; then
              continue
            fi

            # 提取镜像源地址
            source=$(echo "$image" | cut -d'/' -f1)
            if [[ -z "$source" ]]; then
              echo "Skipping invalid image: $image"
              continue
            fi

            # 将镜像添加到对应的文件中
            echo "$image" >> "temp/mirror-list-$source.txt"
            image_lists["$source"]=1
          done < mirror-list.txt

          # 输出拆分后的文件列表
          image_list_files=$(printf "%s\n" "${!image_lists[@]}" | tr '\n' ',')
          echo "Image list files: $image_list_files"
          echo "::set-output name=image-list-files::$image_list_files"
        shell: /usr/bin/bash -e

  sync-images:
    needs: prepare-mirror-lists
    runs-on: ubuntu-latest
    strategy:
      matrix:
        source: ${{ fromJSON(needs.prepare-mirror-lists.outputs.image-list-files) }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Install Skopeo
        run: |
          sudo apt-get update && sudo apt-get install -y skopeo

      - name: Login to private registry
        env:
          PRIVATE_REGISTRY: ${{ secrets.PRIVATE_REGISTRY_HW }}
          PRIVATE_REPOSITORY: ${{ secrets.PRIVATE_REPOSITORY_HW }}
          PRIVATE_REGISTRY_USER: ${{ secrets.PRIVATE_REGISTRY_USER_HW }}
          PRIVATE_REGISTRY_PASSWORD: ${{ secrets.PRIVATE_REGISTRY_PASSWORD_HW }}

        run: |
          echo "$PRIVATE_REGISTRY_PASSWORD" | skopeo login --username="$PRIVATE_REGISTRY_USER" --password-stdin "$PRIVATE_REGISTRY"

      - name: Sync images with Skopeo
        env:
          PRIVATE_REGISTRY: ${{ secrets.PRIVATE_REGISTRY_HW }}
          PRIVATE_REPOSITORY: ${{ secrets.PRIVATE_REPOSITORY_HW }}
        run: |
          # 检查 PRIVATE_REGISTRY 和 PRIVATE_REPOSITORY 是否为空
          if [ -z "$PRIVATE_REGISTRY" ]; then
            echo "Error: PRIVATE_REGISTRY is not set."
            exit 1
          fi
          if [ -z "$PRIVATE_REPOSITORY" ]; then
            echo "Error: PRIVATE_REPOSITORY is not set."
            exit 1
          fi

          # 确定镜像列表文件
          image_list_file="temp/mirror-list-${{ matrix.source }}.txt"

          # 检查镜像列表文件是否存在
          if [ ! -f "$image_list_file" ]; then
            echo "Error: Image list file $image_list_file not found."
            exit 1
          fi

          # 读取镜像列表文件中的每一行镜像名称
          while IFS= read -r image; do
            # 删除前后空格
            image=$(echo "$image" | xargs)
            # 去除多余的斜杠
            image=$(echo "$image" | tr -s '/')
            # 跳过空行和注释行
            if [[ -z "$image" || "$image" =~ ^# ]]; then
              echo "Skipping invalid image: $image"
              continue
            fi

            # 打印当前执行的镜像
            echo "Processing image: $image"

            # 生成目标镜像名称
            target_image="$PRIVATE_REGISTRY/$PRIVATE_REPOSITORY/$image"

            # 打印目标镜像名称
            echo "Target image: $target_image"

            # 使用 Skopeo 复制镜像
            skopeo copy --all "docker://$image" "docker://$target_image"
            echo "Synced $image to $target_image"
          done < "$image_list_file"
        shell: /usr/bin/bash -e