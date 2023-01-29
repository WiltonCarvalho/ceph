# ceph single node
```
multipass launch -n ceph -v -c 4 -m 4G -d 20G --cloud-init multipass.yaml 22.04
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/ceph-osd0.qcow2 10G
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/ceph-osd1.qcow2 10G
virsh attach-disk ceph /var/lib/libvirt/images/ceph-osd0.qcow2 vdc --persistent --subdriver qcow2 --live
virsh attach-disk ceph /var/lib/libvirt/images/ceph-osd1.qcow2 vdd --persistent --subdriver qcow2 --live

multipass shell ceph
```
```
apt update
apt install podman cephadm

cephadm bootstrap \
--cluster-network 192.168.122.0/24 \
--mon-ip $(ip route get 1 | awk '{print $7;exit}') \
--dashboard-password-noupdate \
--initial-dashboard-user admin \
--initial-dashboard-password ceph \
--allow-fqdn-hostname \
--single-host-defaults

cat <<'EOF'> grafana.yml
service_type: grafana
spec:
  initial_admin_password: ceph
EOF

cephadm shell --mount grafana.yml:/var/lib/ceph/grafana.yml
ceph orch apply -i /var/lib/ceph/grafana.yml
ceph orch redeploy grafana
exit

apt install ceph-common

ceph -s
```
```
ceph orch apply osd --all-available-devices

# OR

ceph orch daemon add osd ceph:/dev/vdc
ceph orch daemon add osd ceph:/dev/vdd
```
```
ceph osd pool create rbd
ceph osd pool application enable rbd rbd
rbd create disk1 --size 1G
rbd ls -l

radosgw-admin realm create --rgw-realm=default --default
radosgw-admin zonegroup create --rgw-zonegroup=default --master --default
radosgw-admin zone create --rgw-zonegroup=default --rgw-zone=us-east-1 --master --default

ceph orch host ls
ceph orch apply rgw default us-east-1 --placement="1 ceph"
ceph -s

apt install curl unzip
curl -fsSL https://awscli.amazonaws.com/awscli-exe-linux-$(uname -m).zip -o /tmp/awscliv2.zip
unzip -q /tmp/awscliv2.zip -d /opt
/opt/aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
rm /tmp/awscliv2.zip
rm -rf /opt/aws
aws --version

radosgw-admin user create --uid=test --display-name=test --access-key=test --secret-key=test
aws configure set aws_access_key_id test --profile default
aws configure set aws_secret_access_key test --profile default
aws configure set region us-east-1 --profile default

aws s3 mb s3://test --endpoint-url http://ceph
aws s3 cp /etc/hosts s3://test --endpoint-url http://ceph
aws s3 ls --endpoint-url http://ceph
aws s3 ls s3://test --endpoint-url http://ceph
```
```
apt install libvirt-daemon-system libvirt-clients libvirt-daemon-driver-storage-rbd \
  ovmf qemu-kvm qemu-utils virtinst --no-install-recommends
usermod -aG libvirt,kvm ubuntu
```
