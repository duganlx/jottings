# gitlab 访问权限调研

Project Access Tokens

Generate project access tokens scoped to this project for your applications that need access to the GitLab API.
You can also use project access tokens with Git to authenticate over HTTP(S).

它有五种角色类型，分别是 Guest、Reporter、Developer、Maintainer、Owner。项目授权的范围有六种，分别是 api、read_api、read_registry、write_registry、read_repository、write_repository。具体说明如下

- api: Grants complete read and write access to the scoped project API, including the Package Registry.
- read_api: Grants read access to the scoped project API, including the Package Registry.
- read_registry: Grants read access (pull) to the Container Registry images if a project is private and authorization is required.
- write_registry: Grants write access (push) to the Container Registry.
- read_repository: Grants read access (pull) to the repository.
- write_repository: Grants read and write access (pull and push) to the repository.

我想要用程序通过 gitlab api 去访问某个仓库的代码，访问链接为 `https://gitlab.jhlfund.com/api/v4/projects/:projectId/repository/files/:filepath/raw`，请求头需要带上 `PRIVATE-TOKEN`。该 token 就是 project assess token 页面生成，其中角色和授权范围测试结果如下

| role       | api | read_api | read_repository | write_repository |
| ---------- | --- | -------- | --------------- | ---------------- |
| Guest      | N   | N        | N               | N                |
| Reporter   | Y   | Y        | Y               | N                |
| Developer  | Y   | Y        | Y               | N                |
| Maintainer | Y   | Y        | Y               | N                |
| Owner      | Y   | Y        | Y               | N                |