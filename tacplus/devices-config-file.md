tac-conf

```

```

ubuntu config
```
cat > ~/.ssh/config << 'EOF'
Host 192.168.10.1
    KexAlgorithms +diffie-hellman-group14-sha1,diffie-hellman-group-exchange-sha1
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
EOF
```

