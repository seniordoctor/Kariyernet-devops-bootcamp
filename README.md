## Kariyer.net DevOps Bootcamp Task Soruları

![soru-3](https://github.com/user-attachments/assets/adec12a9-b808-43b8-83f5-d18f4c35072a)

Çözüm aşamaları:

1- Builder VM'de Kubespray projesini çektik ve inventory/sample/inventory.ini dosyasını konfigure ediyoruz. Buradaki amacımız Ansible ile vermiş olduğumuz IP adresleriyle birlikte SSH ile erişeceğiz. Master ve Worker sunucularının hangileri olacağını belirliyoruz.
```
[all]
master ansible_host=172.24.34.20
worker ansible_host=172.24.34.21


[kube_control_plane]
master

[etcd]
master

[kube_node]
worker

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr

[kube_control_plane:vars]
node_labels={"type":"master"}

[kube_node:vars]
node_labels={"type":"worker"}


[all:vars]
ansible_user='<your-user>'
ansible_port=22
ansible_password=<your-password>
become=true
```
2- Azure DevOps arayüzü üzerinden Pipeline oluşturuyoruz. Pipeline üzerinde yapılacak işlemleri iki farklı stage olarak bölümlüyoruz. Build stage de isminden anlaşıldığı üzere hub.docker üzerinde olan projemizi gösterdik. Repomuz içerisinde yazdığımız Dockerfile ve YAML dosyasına göre Build ve Push işlemlerini Container üzerinde gerçekleştirecektir. Deploy stage de ansible yükleme işlemlerimizi gerçekleştiriyoruz ve ilgili dizine giderek Kubespray içerisinde hazır olarak gelen gereklilikleri indiriyoruz, ansible ile birlikte master ve worker sunucularımıza kurulumları otomatik bir şekilde gerçekleştirilecektir. Sonrasında kontrol edebilmek adına Master sunucumuza giderek "kubectl get nodes" komutunu yazarak görüntüleyebiliriz.

```
trigger:
- cicd # Branch name

pool:
  name: Linux # Deployment groups

stages:
- stage: Build
  jobs:
  - job: BuildAndPushDocker
    steps:
    - task: Docker@2
      inputs:
        containerRegistry: 'docker-registry'
        repository: 'seniordoctor/demo-dotnet'
        command: 'buildAndPush'
        Dockerfile: '**/Dockerfile.linux'
        tags: '$(Build.BuildNumber)'

    - task: CopyFiles@2
      inputs:
        SourceFolder: '$(Build.SourcesDirectory)/deploy'
        Contents: 'demo-dotnet.yml'
        TargetFolder: '$(Build.ArtifactStagingDirectory)'

    - task: PublishBuildArtifacts@1
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)'
        ArtifactName: 'drop'
        publishLocation: 'Container'

- stage: Deploy
  dependsOn: Build
  jobs:
  - job: Install_Kubernetes
    steps:
    - checkout: self
    - script: |
        sudo apt update
        sudo apt install -y python3-pip
        sudo apt install -y ansible && sudo apt-get install sshpass
        cd /home/builder/kubespray/
        pip3 install -U -r requirements.txt
        ansible-playbook -i inventory/sample/inventory.ini cluster.yml
      displayName: 'Deploy Kubernetes Cluster with Kubespray'
```

3- Dilerseniz  aşağıdaki görselde görünen triggers ayarlarından filtreleme yaparak cicd branch'e commit geldiğinde bunun çalışmasını sağlayabilirsiniz. CI tarafını oluşturma işlemimiz tamamlandı.

![pipeline-triggers-ci](https://github.com/user-attachments/assets/443260e5-166c-4556-8dc4-8ad2b44a4c1f)

4- CD Süreci için Release Pipeline oluşturmamız gerekmektedir. Release oluşturma işlemi sonrasında aşağıdaki ekran görüntülerinde bulunan CD Trigger, After Release (her Release işlemi sonrasında otomatik olarak çalışacak, manuel işleme ihtiyacı yok) ve Build branch filter ile filtreleme yapabiliriz.

![cd-trigger](https://github.com/user-attachments/assets/56b55e74-4cf7-4fd5-b54e-958b411da19f)
![release-pipeline-cd](https://github.com/user-attachments/assets/788269b2-0d00-473b-96c7-a9e66d082be9)

CI/CD sürecine uygun şekilde ilerleyen ve Kubernetes kurulumunu otomatize işlemleri tamamlanmıştır.

---

![soru-4](https://github.com/user-attachments/assets/c2114698-f164-45ba-be50-aa2bf8ad2532)

Çözüm aşamaları

1- Ansible playbook ile Master ve Worker sunucuları üzerine hedeflerin tek seferde yapılması adına inventory (Ansible ile işlem yapılacak sunucularımız. Giriş bilgileri için inventory.yaml, servislerin kurulması veya disable edilmesi, hostname değiştirilmesi ve yeni user oluşturulması için forth.yml)

Inventory.yaml
``` 
all:
  hosts:
    master:
      ansible_host: 172.24.34.20
    worker:
      ansible_host: 172.24.34.21
  vars:
    ansible_user: '<your-username>'
    ansible_port: 22
    ansible_password: '<your-password>'
    become: true
    ansible_become_method: sudo
```

forth.yml
```
---
- name: Configure servers
  hosts: all
  become: yes
  tasks:
    - name: Install chrony
      apt:
        name: chrony
        state: present
        update_cache: yes

    - name: Restart chrony
      service:
        name: chrony
        state: restarted

    - name: Disable and stop apparmor
      systemd:
        name: apparmor
        enabled: no
        state: stopped

    - name: Get host IP address   # Sunucunun IP adresini öğrenmek için kullandık.
      command: hostname -I
      register: host_ip

    - name: Set hostname
      hostname:
        name: "kariyer-{{ host_ip.stdout.split(' ')[0] }}"    # host IP adresinden -I parametresi bütün IP adreslerimizi getireceği için [0] index'li olanı alıyoruz ve yeni hostname işlemleri gerçekleştiriliyor.

    - name: Ensure 'kariyer' group exists
      group:
        name: kariyer
        gid: 3434
        state: present

    - name: Create user 'kariyer'
      user:
        name: kariyer
        uid: 3434
        group: kariyer
        state: present
```

2- Playbook ve inventory YAML'larımızı yazdıktan sonra çalıştırmak için komutumuzu çalıştırıyoruz.

``` ansible-playbook forth.yml -i invetory.yaml ```

![ansible-playbook](https://github.com/user-attachments/assets/2030cb7a-2149-49f7-8943-d2830ed084e1)

3- Hata almadığımızdan dolayı kontrol etmek için Master ve Worker sunucumuzda hostnamectl ile kontrolümüzü gerçekleştiriyoruz. Eklenmiş yeni user'ı da görüntüleyebiliriz.

![master-hostname](https://github.com/user-attachments/assets/d5140ec1-be0e-4069-a933-8b0261304807)

![worker-hostname](https://github.com/user-attachments/assets/9a95a2c8-3a04-47ba-bcdb-e8de4ae96b1b)

![master-kariyer-user-group](https://github.com/user-attachments/assets/2178010b-75c2-4996-9b2e-64a324c6ef2b)

![worker-kariyer-user](https://github.com/user-attachments/assets/3f9f447e-c997-4d11-8260-524664ccf746)
