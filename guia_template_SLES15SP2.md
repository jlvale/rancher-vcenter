### Configuração do Cloud Init na VM utilizada como Template:

1. Adicionar o repositório "Public Cloud Module"
```
Efetuar a instalação do Cloud Init e habilitar todas as etapas (init, config e final);
  zypper in cloud-init
  systemctl status cloud-init.service
  systemctl enable cloud-init.service
  systemctl enable cloud-config.service
  systemctl enable cloud-final.service
```
2. Alterar o arquivo "/etc/cloud/cloud.cfg"
```
  No campo datasource_list deverá conter a seguinte configuração:
  datasource_list: [ NoCloud, None ]
  Obs. Na utilização de algum outro tipo de nuvem esta configuração deverá conter o datasource apropriado.
```
3. Alterar (ou criar) o arquivo "/etc/cloud/cloud.cfg.d/99_disable_network_config.cfg"
```
  O arquivo deverá conter apenas a seguinte configuração:
  network: {config: disabled}
```
4. Rebootar a máquina e conferir se todos os serviços do cloud-init estão subindo normalmente.
5. Efetuar a limpeza da máquina antes da conversão em forma de template no VMware
```
  Para a limpeza é necessário executar os seguintes passos:
  truncate -s 0 /etc/machine-id
  rm /var/lib/dbus/machine-id
  ln -s /etc/machine-id /var/lib/dbus/machine-id
  truncate -s 0 /etc/machine-id
  rm /var/lib/dbus/machine-id
  ln -s /etc/machine-id /var/lib/dbus/machine-id
  rm -f /etc/udev/rules.d/70-persistent-net.rules
  touch /etc/udev/rules.d/75-persistent-net-generator.rules
  rm -rf /tmp/* /tmp/.* /var/tmp/* /var/tmp/.* &> /dev/null || true
  rm -rf /var/cache/*/* /var/crash/* /var/lib/systemd/coredump/*
  cloud-init clean -s -l
 ```
 6. Desligar a VM e, apenas se o ambiente não possuir DHCP, habilitar o vApp (via interface do vCenter) antes de transformá-la em template;
