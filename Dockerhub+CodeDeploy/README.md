## ๐CI / CD ์ํคํ์ณ

![image](https://user-images.githubusercontent.com/66551410/137348776-6bdfbebc-b83e-4903-823e-02d1a1f950eb.png)

- afterInstall.bash

```sh
# DockerHub์ ์๋ Image๋ ์ฐจํ GitHub Action์ ์ฌ์ฉํด์ git push๋ฅผ ํ  ๋๋ง๋ค ์๋ก์ด image๋ฅผ build ํ์ฌ ๊ฐฑ์ ์์ผ์ค ๊ฒ์๋๋ค.
# ํ๊ฒฝ๋ณ์๋ .gitignore์ ๋ฑ๋กํ์ฌ GitHub์ ์๋ก๋ ๋์ง ์์ผ๋ฏ๋ก Docker Container๋ฅผ ๊ฐ๋์ํฌ ๋ ๋ฐ๋ก ์ฃผ์ํด์ค๋๋ค.
# ํ๊ฒฝ๋ณ์๊ฐ ๋ด๊ธด ํ์ผ์ ๋ณ๋๋ก EC2 instance ๋ด์ ์ง์  ์์ฑํด์ฃผ์์ผ ํฉ๋๋ค.
```

- CI.yml

```yml
# Docker Image๋ฅผ ์๋์ผ๋ก buildํ๊ณ  pushํฉ๋๋ค.
# Docker Hub ID์ PW๋ GitHub settings์ ์๋ secrets์ ์ ์ฅํด๋ก๋๋ค.
# [Your DockerHub ID]/[Your Repository Name] ์์) dal96k/woomin-facebook
      - name: Docker image build and push
        uses: docker/build-push-action@v1
        with:
          username: ${{ secrets.DOCKER_ID }}
          password: ${{ secrets.DOCKER_PW }}
          repository: [Your DockerHub ID]/[Your Repository Name]
          tags: latest

# Docker Hub์ ์๋ก์ด Image๊ฐ push ์๋ฃ๋๋ฉด CodeDeploy Agent๊ฐ ๋์๋๋๋ก ํฉ๋๋ค.
# --application-name๊ณผ --deployment-group-name์ ์๊น ์์ฑํ์  ์ ํ๋ฆฌ์ผ์ด์ ์ด๋ฆ๊ณผ ๊ทธ๋ฃน ์ด๋ฆ์ผ๋ก ๋์ฒดํ์๋ฉด ๋ฉ๋๋ค.
# [Your GitHub Repository] ์์) Woomin-Jeon/facebook-clone-server
# "commitId=${GITHUB_SHA}" ์ฝ๋๊ฐ ์๋์ผ๋ก ์ต์  commit์ ๋ถ๋ฌ์ต๋๋ค.
# ์๊น ๋ณด๊ดํด๋์๋ AWS_ACCESS_KEY_ID์ AWS_SECRET_ACCESS_KEY๋ GitHub secrets์ ์ ์ฅํด๋ก๋๋ค.
      - name: Trigger the CodeDeploy in EC2 instance
        run: aws deploy --region ap-northeast-2 create-deployment --application-name CodeDeploy-application-example --deployment-config-name CodeDeployDefault.OneAtATime --deployment-group-name CodeDeploy-group-example --github-location repository=[Your GitHub Repository],commitId=${GITHUB_SHA}
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          Default_region_name: ap-northeast-2
```
