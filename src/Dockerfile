##### Dockerfile #####
# Stage 1: Builder (Cài đặt dependencies)
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
# Dùng --omit=dev để chỉ cài đặt các package phục vụ production, bỏ qua devDependencies
RUN npm install --omit=dev

# Stage 2: Production Runtime (Môi trường chạy thực tế)
FROM node:18-alpine

RUN mkdir -p /home/node/app/node_modules && chown -R node:node /home/node/app
WORKDIR /home/node/app

COPY --chown=node:node package*.json ./
USER node

# Copy node_modules đã tối ưu từ builder stage và toàn bộ mã nguồn
COPY --from=builder --chown=node:node /app/node_modules ./node_modules
COPY --chown=node:node . .

EXPOSE 3000
CMD [ "node", "index.js" ]