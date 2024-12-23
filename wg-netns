#!/usr/bin/env python3
from __future__ import annotations
from argparse import ArgumentParser, RawDescriptionHelpFormatter
from pathlib import Path
from typing import Any, Optional
import dataclasses
import getpass
import json
import os
import shlex
import subprocess
import sys

WIREGUARD_DIR = Path('/etc/wireguard')
NETNS_DIR = Path('/etc/netns')
VERBOSE = 0
WG = Path('/usr/bin/wg')
IP = Path('/usr/sbin/ip')


def main():
    try:
        cli(sys.argv[1:])
        sys.exit(0)
    except Exception as e:
        print(f'error: {e} ({e.__class__.__name__})', file=sys.stderr)
        if VERBOSE:
            raise
        sys.exit(1)


def cli(args):
    global WIREGUARD_DIR
    global NETNS_DIR
    global VERBOSE

    entrypoint = ArgumentParser(
        formatter_class=RawDescriptionHelpFormatter,
        epilog=(
            'environment variables:\n'
            f'  WG_PROFILE_DIR      wireguard config dir, default: {WIREGUARD_DIR}\n'
            f'  WG_NETNS_DIR        network namespace config dir, default: {NETNS_DIR}\n'
            f'  WG_VERBOSE          print detailed output if 1, default: {VERBOSE}\n'
        ),
    )

    subparsers = entrypoint.add_subparsers(dest='action', required=True, metavar='ACTION')

    parser = subparsers.add_parser('up', help='setup namespace and associated interfaces')
    parser.add_argument('profile', type=lambda x: Path(x).expanduser(), metavar='PROFILE', help='name or path of profile')

    parser = subparsers.add_parser('down', help='teardown namespace and associated interfaces')
    parser.add_argument('-f', '--force', action='store_true', help='ignore errors')
    parser.add_argument('profile', type=lambda x: Path(x).expanduser(), metavar='PROFILE', help='name or path of profile')

    opts = entrypoint.parse_args(args)

    try:
        WIREGUARD_DIR = Path(os.environ.get('WG_PROFILE_DIR', WIREGUARD_DIR))
        NETNS_DIR = Path(os.environ.get('WG_NETNS_DIR', NETNS_DIR))
        VERBOSE = int(os.environ.get('WG_VERBOSE', VERBOSE))
    except Exception as e:
        raise RuntimeError(f'failed to load environment variable: {e} (e.__class__.__name__)') from e

    if opts.action == 'up':
        namespace = Namespace.from_profile(opts.profile)
        try:
            namespace.setup()
        except KeyboardInterrupt:
            namespace.teardown(check=False)
        except Exception:
            namespace.teardown(check=False)
            raise
    elif opts.action == 'down':
        namespace = Namespace.from_profile(opts.profile)
        namespace.teardown(check=not opts.force)
    else:
        raise RuntimeError('congratulations, you reached unreachable code')


@dataclasses.dataclass
class Peer:
    public_key: str
    preshared_key: Optional[str] = None
    name: Optional[str] = None
    endpoint: Optional[str] = None
    persistent_keepalive: int = 0
    allowed_ips: list[str] = dataclasses.field(default_factory=list)
    routes: Optional[list[str]] = None

    @classmethod
    def from_dict(cls, data: dict[str, Any]) -> Peer:
        data = {key.replace('-', '_'): value for key, value in data.items()}
        return cls(**data)

    def setup(self, interface: Interface, namespace: str|None) -> Peer:
        options = [
            'peer', self.public_key,
            'preshared-key', '/dev/stdin' if self.preshared_key else '/dev/null',
            'persistent-keepalive', self.persistent_keepalive,
        ]
        if self.endpoint:
            options.extend(('endpoint', self.endpoint))
        if self.allowed_ips:
            options.extend(('allowed-ips', ','.join(self.allowed_ips)))
        wg('set', interface.name, *options, stdin=self.preshared_key, netns=namespace)
        return self


@dataclasses.dataclass
class Interface:
    name: str
    base_netns: str|None = None
    private_key: Optional[str] = None
    public_key: Optional[str] = None
    address: list[str] = dataclasses.field(default_factory=list)
    listen_port: int = 0
    fwmark: int = 0
    mtu: int = 1420
    peers: list[Peer] = dataclasses.field(default_factory=list)

    @classmethod
    def from_dict(cls, data: dict[str, Any], base_netns: str|None = None) -> Interface:
        peers = data.pop('peers', list())
        peers = [Peer.from_dict({key.replace('-', '_'): value for key, value in peer.items()}) for peer in peers]
        return cls(**data, peers=peers, base_netns=base_netns)

    def setup(self, namespace: Namespace) -> Interface:
        self._create()
        self._configure_wireguard()
        for peer in self.peers:
            peer.setup(self, self.base_netns)
        self._assign_namespace(namespace.name)
        self._assign_addresses(namespace.name)
        self._bring_up(namespace.name)
        self._create_routes(namespace.name)
        return self

    def _create(self) -> None:
        ip('link', 'add', self.name, 'type', 'wireguard', netns=self.base_netns)

    def _configure_wireguard(self) -> None:
        wg('set', self.name, 'listen-port', self.listen_port, netns=self.base_netns)
        wg('set', self.name, 'fwmark', self.fwmark, netns=self.base_netns)
        if self.private_key:
            wg('set', self.name, 'private-key', '/dev/stdin', stdin=self.private_key, netns=self.base_netns)

    def _assign_namespace(self, namespace: str|None) -> None:
        ip('link', 'set', self.name, 'netns', namespace if namespace else '1', netns=self.base_netns)

    def _assign_addresses(self, namespace: str|None) -> None:
        for address in self.address:
            ip('-6' if ':' in address else '-4', 'address', 'add', address, 'dev', self.name, netns=namespace)

    def _bring_up(self, namespace: str|None) -> None:
        ip('link', 'set', 'dev', self.name, 'mtu', self.mtu, 'up', netns=namespace)

    def _create_routes(self, namespace: str|None):
        for peer in self.peers:
            networks = peer.routes if peer.routes is not None else peer.allowed_ips
            for network in networks:
                ip('-6' if ':' in network else '-4', 'route', 'add', network, 'dev', self.name, netns=namespace)

    def teardown(self, namespace: Namespace, check=True) -> Interface:
        if self.exists(namespace):
            ip('link', 'set', self.name, 'down', check=check, netns=namespace.name)
            ip('link', 'delete', self.name, check=check, netns=namespace.name)
        return self

    def exists(self, namespace: Namespace) -> bool:
        try:
            ip('link', 'show', self.name, capture=True, netns=namespace.name)
            return True
        except Exception:
            return False


@dataclasses.dataclass
class Namespace:
    name: str|None
    managed: bool = True
    dns_server: list[str] = dataclasses.field(default_factory=list)
    interfaces: list[Interface] = dataclasses.field(default_factory=list)

    @classmethod
    def from_profile(cls, path: Path) -> Namespace:
        try:
            return cls.from_dict(cls._read_profile(cls._find_profile(path)))
        except Exception as e:
            raise RuntimeError(f'failed to load profile: {e}') from e

    @staticmethod
    def _find_profile(profile: Path) -> Path:
        if not profile.is_file() and profile.name == profile.as_posix():  # path does not contain '/' and '.'
            for extension in ('yaml', 'yml', 'json'):
                path = WIREGUARD_DIR/f'{profile.name}.{extension}'
                if path.is_file():
                    return path
        return profile

    @staticmethod
    def _read_profile(profile: Path) -> dict[str, Any]:
        with open(profile) as file:
            if profile.suffix in ('.yaml', '.yml'):
                if not YAML_SUPPORTED:
                    raise RuntimeError(f'can not load profile in yaml format if pyyaml library is not installed')
                return yaml.safe_load(file)
            elif profile.suffix == '.json':
                return json.load(file)
            else:
                raise RuntimeError(f'unsupported file format {profile.suffix.removeprefix(".")}')

    @classmethod
    def from_dict(cls, data: dict[str, Any]) -> Namespace:
        data = {key.replace('-', '_'): value for key, value in data.items()}
        scriptlets = {key: data.pop(key, None) for key in ['pre_up', 'post_up', 'pre_down', 'post_down']}
        scriptlets = {key: Scriptlet.from_value(value) for key, value in scriptlets.items() if value is not None}
        interfaces = data.pop('interfaces', list())
        base_netns = data.pop('base_netns', None)
        interfaces = [Interface.from_dict({key.replace('-', '_'): value for key, value in interface.items()}, base_netns=base_netns) for interface in interfaces]
        return cls(**data, **scriptlets, interfaces=interfaces)  # type: ignore

    def setup(self) -> Namespace:
        if self.managed and self.name:
            self._create()
            self._write_resolvconf()
        for interface in self.interfaces:
            interface.setup(self)
        return self

    def teardown(self, check=True) -> Namespace:
        for interface in self.interfaces:
            interface.teardown(self, check=check)
        if self.managed and self.exists():
            self._delete(check)
            self._delete_resolvconf()
        return self

    def exists(self) -> bool:
        namespaces = json.loads(ip('-j', 'netns', 'list', capture=True))
        return self.name in {namespace['name'] for namespace in namespaces}

    def _create(self) -> None:
        ip('netns', 'add', self.name)
        ip('link', 'set', 'dev', 'lo', 'up', netns=self.name)

    def _delete(self, check=True) -> None:
        ip('netns', 'delete', self.name, check=check)

    @property
    def _resolvconf_path(self) -> Path:
        assert self.name
        return NETNS_DIR/self.name/'resolv.conf'

    def _write_resolvconf(self) -> None:
        if self.dns_server:
            self._resolvconf_path.parent.mkdir(parents=True, exist_ok=True)
            content = '\n'.join(f'nameserver {server}' for server in self.dns_server)
            self._resolvconf_path.write_text(content)

    def _delete_resolvconf(self) -> None:
        if self._resolvconf_path.exists():
            self._resolvconf_path.unlink()
        try:
            NETNS_DIR.rmdir()
        except OSError:
            pass


def wg(*args, netns: str|None = None, stdin: str|None = None, check=True, capture=False) -> str:
    return run(WG, *args, stdin=stdin, check=check, capture=capture)


def ip(*args, stdin: str|None = None, netns: str|None =None, check=True, capture=False) -> str:
    return run(IP, *(['-n', netns] if netns else []), *args, stdin=stdin, check=check, capture=capture)


def run(*args, stdin: str|None = None, check=True, capture=False) -> str:
    args = [str(item) for item in args if item is not None]
    if VERBOSE:
        print('>', ' '.join(args), file=sys.stderr)
    process = subprocess.run(args, input=stdin, text=True, capture_output=capture)
    if check and process.returncode != 0:
        error = process.stderr.strip() if process.stderr else f'exit code {process.returncode}'
        raise RuntimeError(f'subprocess failed: {" ".join(args)}: {error}')
    return process.stdout


if __name__ == '__main__':
    main()
