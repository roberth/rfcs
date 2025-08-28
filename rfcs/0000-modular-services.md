---
feature: modular-services
start-date: 2025-08-28
author: @roberth
co-authors: (to be determined)
shepherd-team: (to be nominated and accepted by RFC steering committee)
shepherd-leader: (to be appointed by RFC steering committee)
related-issues: |
  - RFC 163: https://github.com/NixOS/rfcs/pull/163
  - Implementation: https://github.com/NixOS/nixpkgs/pull/372170
  - Tracking: https://github.com/NixOS/nixpkgs/issues/428084
---

# Summary
[summary]: #summary

Standardize the modular services system in NixOS to enable multi-instance service deployment, service composition, and cross-platform portability through module-based service definitions that complement traditional NixOS services.

# Status

**READ THIS:**

This Status section will be removed before RFC acceptance, and serves to manage expectations.
Modular Services was merged into NixOS on an experimental basis, and its acceptance is conditional on community support and continued development.
This RFC will be updated as the project progresses, and serves a dual purpose as an introduction to the project while development is ongoing.

<!-- end of status section which will be removed -->

# Motivation
[motivation]: #motivation

Traditional NixOS services work well for system infrastructure but cannot address several emerging use cases. Multi-instance deployment requires running multiple copies of the same service type with different configurations - impossible with singleton services like `services.nginx`. Service composition demands reusable components that can be combined into complex applications - difficult with monolithic service definitions. Package-service integration needs service definitions that ship with packages to ensure version compatibility - not supported by separate service modules.

The modular services system addresses these problems through a working implementation merged in July 2025. Real services like ghostunnel and PHP-FPM demonstrate practical usability, while external projects like NixNG and home-manager express interest in adoption. The module-based architecture leverages familiar NixOS patterns, enabling gradual adoption without disrupting existing workflows.

This RFC standardizes the experimental implementation to provide service authors with stable patterns for NixOS integration, configuration management, and multi-platform deployment. The goal is enabling new capabilities while maintaining coexistence with traditional services.

# Detailed design
[design]: #detailed-design

## Service Definition Structure

Modular services are NixOS modules with mandatory `_class = "service"` declaration. Services define portable configuration through standard module options and extend with service-manager-specific features using conditional patterns.

Service modules declare process execution through `process.argv` option, provide configuration files via `configData` system, and integrate with NixOS through optional capability interfaces.

## Multi-Instance System

The `system.services.<name>` attribute tree enables arbitrary service instantiation. Each service can define sub-services through nested `services` options, creating hierarchical compositions that map to unique systemd units.

## Service Manager Abstraction

Services target multiple platforms through conditional configuration. Portable sections work across all service managers while platform-specific sections use `lib.optionalAttrs (options ? systemd)` guards. Service managers provide appropriate option structures and handle deployment-specific integration.

## NixOS Integration Interface

Services interact with NixOS subsystems through standardized capability declarations. The `nixos` namespace provides options for user creation, firewall management, file placement, package installation, and kernel configuration. SystemD integration layer maps these declarations to appropriate NixOS modules when present.

## Configuration Data System

Services manage dynamic configuration through the `configData` interface. Files declared with reload capabilities can be updated without service restart. The system maps configuration to `/etc/system-services/<service-name>/` paths with hierarchical organization for nested services.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

## Complete Service Implementation

```nix
# myapp-service.nix
{ lib, config, writeText, ... }:
{
  _class = "service";
  
  options = {
    package = lib.mkOption {
      type = lib.types.package;
      description = "The myapp-service package to use.";
      defaultText = lib.literalMD "The package that carried this service module.";
    };
    port = lib.mkOption {
      type = lib.types.port;
      default = 8080;
    };
    user = lib.mkOption {
      type = lib.types.str;
      default = "myapp";
    };
  };
  
  config = {
    process.argv = [ 
      "${config.package}/bin/myapp" 
      "--port" (toString config.port)
      "--config" "${config.configData."app.conf".path}"
    ];
    
    configData."app.conf" = {
      content = writeText "app.conf" ''
        port=${toString config.port}
        user=${config.user}
      '';
    };
    
    # NixOS integration capabilities
    nixos = {
      users.${config.user} = {
        isSystemUser = true;
        group = config.user;
      };
      firewall.allowedTCPPorts = [ config.port ];
    };
    
    meta.maintainers = [ lib.maintainers.example ];
  } // lib.optionalAttrs (options ? systemd) {
    systemd.services."" = {
      wants = [ "network-online.target" ];
      after = [ "network-online.target" ];
    };
  };
}
```

## Package Integration

```nix
# Package provides service definition
stdenv.mkDerivation (finalAttrs: {
  # ... package definition ...
  
  passthru.services.default = {
    imports = [
      (lib.modules.importApply ./myapp-service.nix {
        inherit writeText;
      })
    ];
    myapp.package = finalAttrs.finalPackage;
  };
})
```

## Multi-Instance Deployment

```nix
{ pkgs, ... }: {
  # This is a regular NixOS module
  _class = "nixos";

  # System nginx separate from GitLab's embedded nginx
  system.services.nginx = {
    imports = [ pkgs.nginx.services.default ];
    nginx = {
      port = 80;
      config = ./system-nginx.conf;
    };
  };
  
  # GitLab with embedded multi-service architecture
  system.services.gitlab = {
    # This `imports` would be sufficient for loading our GitLab distribution
    imports = [ pkgs.gitlab.services.default ];
    
    # GitLab's own nginx instance for application routing
    services.nginx = {
      # This sub-service and most of its configuration is already provided by
      # pkgs.gitlab.services.default
      # Unlike a traditional NixOS service, we can have this extra nginx instance
      # without any conflicts.
      # imports = [ pkgs.nginx.services.default ];
      nginx = {
        port = 8080;
      };
    };
    
    # Git repository storage service
    services.gitaly = {
      gitaly = {
        port = 8075;
        storageDir = "/var/lib/gitlab/repositories";
      };
    };

    # Rails application server
    # services.rails = ...;
    # ... etc ...
  };
}
```

## Cross-Platform Service

```nix
{
  _class = "service";
  
  config = {
    # Always portable
    process.argv = [ "${package}/bin/server" ];
    meta.maintainers = [ maintainers.example ];
    
    # Conditional platform features
  } // lib.optionalAttrs (options ? systemd) {
    systemd.services."" = {
      # SystemD-specific configuration
    };
  } // lib.optionalAttrs (options ? launchd) {
    launchd.agents."" = {
      # macOS launchd configuration
    };
  };
}
```

These examples demonstrate the key interactions: service definition, package integration, multi-instance deployment, service composition, and cross-platform targeting. The system enables complex deployments while maintaining familiar NixOS module patterns.

# Drawbacks
[drawbacks]: #drawbacks

Modular services add conceptual complexity to NixOS by introducing a parallel service system alongside traditional services. Service authors must learn additional patterns and make decisions about which approach to use for their services.

Poorly written service definitions could create security vulnerabilities or system instability. Modular services are *not* a security mechanism that provides any kind isolation. The NixOS integration could be extended to enforce security rules at evaluation time.

Performance impact from module system evaluation overhead may become significant with large numbers of modular services, though this remains untested at such a scale. This is mitigated in the following ways:
  - Only the services that are used are loaded and evaluated. This results in a speedup as the number of traditional services is reduced. Not loading and partially evaluating all possible service types has been [shown](https://github.com/NixOS/nixpkgs/issues/137168#issue-991981735) to lead to a 2 to 4 times speedup in evaluation time.
  - The portability aspect can be leveraged to deploy services in a non-atomic way, which may be more appropriate at the potentially extreme scale where this performance is expected to be problematic. A non-atomic deployment, could be performed with systemd units in `/run/systemd`, and the evaluation required for that would be limited to the units that are put into place.
    Note that the cost of starting or switching a service will typically be significantly higher than its module evaluation cost.

# Alternatives
[alternatives]: #alternatives

## Status Quo - Traditional Services Only

Continuing with only `services.*` modules would maintain simplicity but leaves multi-instance and composition use cases unsolved. Projects needing these capabilities must implement ad-hoc solutions or abandon NixOS for deployment.

## Function-Based Approach (RFC 163)

The original RFC 163 proposed function-based service creation with explicit backend selection. This approach failed to gain community acceptance due to departure from familiar NixOS patterns and implementation complexity. The module-based approach succeeded by building on established foundation.

## Container-Based Deployment

Using containers for multi-instance deployment provides isolation but sacrifices NixOS's declarative advantages and requires separate container orchestration. Modular services preserve declarative configuration while enabling similar deployment flexibility.

NixOS Containers being whole NixOS configurations exacerbates the evaluation performance.

## External Service Managers

Adopting external service managers like Docker Compose or Kubernetes would provide advanced orchestration but break integration with NixOS system management. Modular services extend NixOS capabilities rather than replacing them. It keeps host deployments predictable; the main strength of NixOS.

# Prior art
[prior-art]: #prior-art

## RFC 163: Portable Service Layer

RFC 163 (October 2023 - January 2025) identified the same problems addressed by modular services but proposed a function-based solution. The extensive 15-month discussion (192 comments) revealed community preference for module-based approaches and highlighted the importance of concrete implementations over theoretical designs. Key insights from the failed RFC informed the successful modular services implementation.

## Docker Compose and Kubernetes

Container orchestration systems demonstrate the value of service composition and multi-instance deployment. Docker Compose uses YAML service definitions that can be composed into complex applications, while Kubernetes provides pod specifications that can be instantiated multiple times. These systems validate the core concepts but lack the deep system integration and type safety that NixOS provides.

## Nix-Darwin and Home-Manager

Cross-platform Nix deployment tools face similar challenges with service portability. Home-manager issue #4751 specifically requests "non-systemd service manager support" indicating broader ecosystem need. NixNG issue #67 expresses interest in accessing NixOS's "much bigger service pool" through modular services, demonstrating external validation of the approach.

## Module System

The success of NixOS modules for system configuration provides precedent for module-based service definitions.

It has been adopted by various configuration managers, like aforementioned Nix-Darwin and Home-Manager, as well as Arion (for Docker Compose) and flake-parts for Nix Flakes.

These projects validate the broader applicability of module-based approaches in the Nix ecosystem.

The modular services implementation builds on old, as well as recent patterns, such as
- Submodules, from ancient NixOS times
- `shorthandOnlyDefinesConfig = false;` to make `imports` into submodules easier; originally for Arion
- RFC 42 `freeformType` for open ended service settings, originally for NixOS
- `_class`, originally added for flake-parts, which deals in all kinds of modules

# Unresolved questions
[unresolved]: #unresolved-questions

The scope and interface design for NixOS integration capabilities requires community input. Which subsystems should be included in the initial standardization, and what level of abstraction provides the right balance between functionality and simplicity?

Service manager portability patterns need validation beyond systemd. What constitutes a minimum viable interface for alternative service managers, and how should services handle functionality gaps between different platforms?

Configuration management conventions may benefit from broader ecosystem examples. Are there patterns across different service types that should be standardized, and how should secrets and runtime updates be handled consistently? RFC 189 appears instrumental in formalizing such abstractions and references.

# Future work
[future]: #future-work

Implementation of alternative service manager backends would validate the portability abstraction and enable broader ecosystem adoption. Nix-darwin integration through launchd support and container deployment through runit or OpenRC would demonstrate cross-platform capabilities.

Integration with RFC 189 (Contracts) could enhance modular services through structured service-to-service interfaces, providing complementary functionality for complex service composition and dependency management.

Tooling development would ease adoption through service definition templates, validation utilities, and documentation generation. Integration with search.nixos.org would improve service discoverability and ecosystem growth.

Migration documentation and examples would accelerate adoption by providing clear patterns for converting existing services and integrating modular services into established workflows.
