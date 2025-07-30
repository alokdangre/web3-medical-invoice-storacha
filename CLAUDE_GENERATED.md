# Claude Generated Code

**Task**: Objective: Help design and implement new features based on the current roadmap or project needs. Focus on clean, scalable, and testable code  Claude Tasks:  🧩 Feature Implementation Read the existing codebase and integrate new feature modules where appropriate.  Ensure new code follows existing patterns and conventions.  🗂️ Example Features (replace/add specifics as needed): Add user role-based access control  Implement export to PDF  Add multi-language support  Enable dark mode toggle  🧪 Testing & Documentation Add unit/integration tests for new functionality.  Update README or in-code comments if required.

**Generated on**: Wed Jul 30 20:02:36 UTC 2025

## Generated Content

```
Based on the current codebase context, I'll implement a comprehensive user role-based access control system for the web3 medical invoice storage application. This feature will integrate with the existing web3 infrastructure and provide secure, role-based permissions.

## FILES_TO_CREATE

### 1. `src/types/auth.ts`
### 2. `src/hooks/useAuth.ts`
### 3. `src/hooks/useRole.ts`
### 4. `src/contexts/AuthContext.tsx`
### 5. `src/components/auth/RoleGuard.tsx`
### 6. `src/components/auth/LoginForm.tsx`
### 7. `src/components/auth/RoleSelector.tsx`
### 8. `src/utils/roleUtils.ts`
### 9. `src/services/authService.ts`
### 10. `src/__tests__/components/RoleGuard.test.tsx`
### 11. `src/__tests__/hooks/useAuth.test.tsx`
### 12. `src/__tests__/utils/roleUtils.test.tsx`

## FILES_TO_MODIFY

### 1. `src/App.tsx` (assuming it exists)
### 2. `package.json`

## CODE_CHANGES

### FILES_TO_CREATE

**src/types/auth.ts**
```typescript
export enum UserRole {
  PATIENT = 'patient',
  HEALTHCARE_PROVIDER = 'healthcare_provider',
  INSURANCE_COMPANY = 'insurance_company',
  ADMIN = 'admin',
  AUDITOR = 'auditor'
}

export enum Permission {
  // Invoice permissions
  CREATE_INVOICE = 'create_invoice',
  VIEW_OWN_INVOICES = 'view_own_invoices',
  VIEW_ALL_INVOICES = 'view_all_invoices',
  EDIT_INVOICE = 'edit_invoice',
  DELETE_INVOICE = 'delete_invoice',
  
  // User management
  MANAGE_USERS = 'manage_users',
  VIEW_USER_PROFILES = 'view_user_profiles',
  
  // System administration
  SYSTEM_ADMIN = 'system_admin',
  AUDIT_LOGS = 'audit_logs',
  
  // Export/Import
  EXPORT_DATA = 'export_data',
  IMPORT_DATA = 'import_data'
}

export interface User {
  id: string;
  walletAddress: string;
  role: UserRole;
  email?: string;
  name?: string;
  organizationId?: string;
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  error: string | null;
}

export interface RolePermissions {
  [key: string]: Permission[];
}

export interface AuthContextType extends AuthState {
  login: (walletAddress: string, signature: string) => Promise<void>;
  logout: () => void;
  hasPermission: (permission: Permission) => boolean;
  hasAnyPermission: (permissions: Permission[]) => boolean;
  updateUserRole: (role: UserRole) => Promise<void>;
}
```

**src/utils/roleUtils.ts**
```typescript
import { UserRole, Permission, RolePermissions } from '../types/auth';

export const ROLE_PERMISSIONS: RolePermissions = {
  [UserRole.PATIENT]: [
    Permission.VIEW_OWN_INVOICES,
    Permission.EXPORT_DATA
  ],
  [UserRole.HEALTHCARE_PROVIDER]: [
    Permission.CREATE_INVOICE,
    Permission.VIEW_OWN_INVOICES,
    Permission.EDIT_INVOICE,
    Permission.EXPORT_DATA,
    Permission.VIEW_USER_PROFILES
  ],
  [UserRole.INSURANCE_COMPANY]: [
    Permission.VIEW_ALL_INVOICES,
    Permission.EXPORT_DATA,
    Permission.AUDIT_LOGS
  ],
  [UserRole.AUDITOR]: [
    Permission.VIEW_ALL_INVOICES,
    Permission.AUDIT_LOGS,
    Permission.EXPORT_DATA
  ],
  [UserRole.ADMIN]: [
    Permission.CREATE_INVOICE,
    Permission.VIEW_ALL_INVOICES,
    Permission.EDIT_INVOICE,
    Permission.DELETE_INVOICE,
    Permission.MANAGE_USERS,
    Permission.VIEW_USER_PROFILES,
    Permission.SYSTEM_ADMIN,
    Permission.AUDIT_LOGS,
    Permission.EXPORT_DATA,
    Permission.IMPORT_DATA
  ]
};

export const getRolePermissions = (role: UserRole): Permission[] => {
  return ROLE_PERMISSIONS[role] || [];
};

export const hasPermission = (userRole: UserRole, permission: Permission): boolean => {
  const permissions = getRolePermissions(userRole);
  return permissions.includes(permission);
};

export const hasAnyPermission = (userRole: UserRole, permissions: Permission[]): boolean => {
  return permissions.some(permission => hasPermission(userRole, permission));
};

export const getRoleDisplayName = (role: UserRole): string => {
  const roleNames = {
    [UserRole.PATIENT]: 'Patient',
    [UserRole.HEALTHCARE_PROVIDER]: 'Healthcare Provider',
    [UserRole.INSURANCE_COMPANY]: 'Insurance Company',
    [UserRole.ADMIN]: 'Administrator',
    [UserRole.AUDITOR]: 'Auditor'
  };
  return roleNames[role] || role;
};

export const canAccessResource = (
  userRole: UserRole,
  resourceOwnerId: string,
  currentUserId: string,
  requiredPermission: Permission
): boolean => {
  // Admin and auditor can access all resources
  if (hasPermission(userRole, Permission.VIEW_ALL_INVOICES)) {
    return true;
  }
  
  // Users can access their own resources if they have the permission
  if (resourceOwnerId === currentUserId && hasPermission(userRole, requiredPermission)) {
    return true;
  }
  
  return false;
};
```

**src/services/authService.ts**
```typescript
import { User, UserRole } from '../types/auth';

export class AuthService {
  private static readonly STORAGE_KEY = 'medical_invoice_auth';
  private static readonly API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:3001';

  static async authenticateWallet(walletAddress: string, signature: string): Promise<User> {
    try {
      const response = await fetch(`${this.API_BASE_URL}/auth/wallet`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          walletAddress,
          signature
        }),
      });

      if (!response.ok) {
        throw new Error(`Authentication failed: ${response.statusText}`);
      }

      const userData = await response.json();
      this.storeAuthData(userData);
      return userData;
    } catch (error) {
      console.error('Authentication error:', error);
      throw new Error('Failed to authenticate wallet');
    }
  }

  static async registerUser(
    walletAddress: string,
    role: UserRole,
    additionalData: Partial<User> = {}
  ): Promise<User> {
    try {
      const response = await fetch(`${this.API_BASE_URL}/auth/register`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          walletAddress,
          role,
          ...additionalData
        }),
      });

      if (!response.ok) {
        throw new Error(`Registration failed: ${response.statusText}`);
      }

      const userData = await response.json();
      this.storeAuthData(userData);
      return userData;
    } catch (error) {
      console.error('Registration error:', error);
      throw new Error('Failed to register user');
    }
  }

  static async updateUserRole(userId: string, newRole: UserRole): Promise<User> {
    try {
      const authToken = this.getAuthToken();
      const response = await fetch(`${this.API_BASE_URL}/users/${userId}/role`, {
        method: 'PUT',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${authToken}`
        },
        body: JSON.stringify({ role: newRole }),
      });

      if (!response.ok) {
        throw new Error(`Role update failed: ${response.statusText}`);
      }

      const userData = await response.json();
      this.storeAuthData(userData);
      return userData;
    } catch (error) {
      console.error('Role update error:', error);
      throw new Error('Failed to update user role');
    }
  }

  static getStoredAuth(): User | null {
    try {
      const stored = localStorage.getItem(this.STORAGE_KEY);
      if (!stored) return null;
      
      const parsed = JSON.parse(stored);
      return {
        ...parsed,
        createdAt: new Date(parsed.createdAt),
        updatedAt: new Date(parsed.updatedAt)
      };
    } catch (error) {
      console.error('Error parsing stored auth data:', error);
      this.clearAuthData();
      return null;
    }
  }

  static storeAuthData(user: User): void {
    try {
      localStorage.setItem(this.STORAGE_KEY, JSON.stringify(user));
    } catch (error) {
      console.error('Error storing auth data:', error);
    }
  }

  static clearAuthData(): void {
    localStorage.removeItem(this.STORAGE_KEY);
  }

  static getAuthToken(): string | null {
    const user = this.getStoredAuth();
    return user?.id || null;
  }

  static async logout(): Promise<void> {
    try {
      const authToken = this.getAuthToken();
      if (authToken) {
        await fetch(`${this.API_BASE_URL}/auth/logout`, {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${authToken}`
          }
        });
      }
    } catch (error) {
      console.error('Logout error:', error);
    } finally {
      this.clearAuthData();
    }
  }
}
```

**src/contexts/AuthContext.tsx**
```typescript
import React, { createContext, useContext, useReducer, useEffect, ReactNode } from 'react';
import { AuthState, AuthContextType, User, UserRole, Permission } from '../types/auth';
import { AuthService } from '../services/authService';
import { hasPermission, hasAnyPermission } from '../utils/roleUtils';

interface AuthAction {
  type: 'SET_LOADING' | 'SET_USER' | 'SET_ERROR' | 'LOGOUT';
  payload?: any;
}

const initialState: AuthState = {
  user: null,
  isAuthenticated: false,
  isLoading: true,
  error: null,
};

const authReducer = (state: AuthState, action: AuthAction): AuthState => {
  switch (action.type) {
    case 'SET_LOADING':
      return { ...state, isLoading: action.payload, error: null };
    
    case 'SET_USER':
      return {
        ...state,
        user: action.payload,
        isAuthenticated: !!action.payload,
        isLoading: false,
        error: null,
      };
    
    case 'SET_ERROR':
      return {
        ...state,
        error: action.payload,
        isLoading: false,
      };
    
    case 'LOGOUT':
      return {
        ...initialState,
        isLoading: false,
      };
    
    default:
      return state;
  }
};

const AuthContext = createContext<AuthContextType | undefined>(undefined);

interface AuthProviderProps {
  children: ReactNode;
}

export const AuthProvider: React.FC<AuthProviderProps> = ({ children }) => {
  const [state, dispatch] = useReducer(authReducer, initialState);

  useEffect(() => {
    // Check for stored authentication on app load
    const storedUser = AuthService.getStoredAuth();
    if (storedUser) {
      dispatch({ type: 'SET_USER', payload: storedUser });
    } else {
      dispatch({ type: 'SET_LOADING', payload: false });
    }
  }, []);

  const login = async (walletAddress: string, signature: string): Promise<void> => {
    dispatch({ type: 'SET_LOADING', payload: true });
    
    try {
      const user = await AuthService.authenticateWallet(walletAddress, signature);
      dispatch({ type: 'SET_USER', payload: user });
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : 'Login failed';
      dispatch({ type: 'SET_ERROR', payload: errorMessage });
      throw error;
    }
  };

  const logout = async (): Promise<void> => {
    dispatch({ type: 'SET_LOADING', payload: true });
    
    try {
      await AuthService.logout();
    } catch (error) {
      console.error('Logout error:', error);
    } finally {
      dispatch({ type: 'LOGOUT' });
    }
  };

  const updateUserRole = async (role: UserRole): Promise<void> => {
    if (!state.user) throw new Error('No user logged in');
    
    dispatch({ type: 'SET_LOADING', payload: true });
    
    try {
      const updatedUser = await AuthService.updateUserRole(state.user.id, role);
      dispatch({ type: 'SET_USER', payload: updatedUser });
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : 'Role update failed';
      dispatch({ type: 'SET_ERROR', payload: errorMessage });
      throw error;
    }
  };

  const hasUserPermission = (permission: Permission): boolean => {
    if (!state.user) return false;
    return hasPermission(state.user.role, permission);
  };

  const hasAnyUserPermission = (permissions: Permission[]): boolean => {
    if (!state.user) return false;
    return hasAnyPermission(state.user.role, permissions);
  };

  const contextValue: AuthContextType = {
    ...state,
    login,
    logout,
    hasPermission: hasUserPermission,
    hasAnyPermission: hasAnyUserPermission,
    updateUserRole,
  };

  return (
    <AuthContext.Provider value={contextValue}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuthContext = (): AuthContextType => {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuthContext must be used within an AuthProvider');
  }
  return context;
};
```

**src/hooks/useAuth.ts**
```typescript
import { useCallback } from 'react';
import { useAuthContext } from '../contexts/AuthContext';
import { useAccount, useSignMessage } from 'wagmi';
import { UserRole } from '../types/auth';
import { AuthService } from '../services/authService';

export const useAuth = () => {
  const authContext = useAuthContext();
  const { address, isConnected } = useAccount();
  const { signMessageAsync } = useSignMessage();

  const loginWithWallet = useCallback(async () => {
    if (!isConnected || !address) {
      throw new Error('Wallet not connected');
    }

    try {
      // Create a message to sign for authentication
      const message = `Sign this message to authenticate with Medical Invoice Storage\nAddress: ${address}\nTimestamp: ${Date.now()}`;
      
      const signature = await signMessageAsync({ message });
      await authContext.login(address, signature);
    } catch (error) {
      console.error('Wallet login failed:', error);
      throw error;
    }
  }, [address, isConnected, signMessageAsync, authContext]);

  const registerWithWallet = useCallback(async (
    role: UserRole,
    additionalData: { email?: string; name?: string; organizationId?: string } = {}
  ) => {
    if (!isConnected || !address) {
      throw new Error('Wallet not connected');
    }

    try {
      await AuthService.registerUser(address, role, additionalData);
      await loginWithWallet();
    } catch (error) {
      console.error('Wallet registration failed:', error);
      throw error;
    }
  }, [address, isConnected, loginWithWallet]);

  return {
    ...authContext,
    walletAddress: address,
    isWalletConnected: isConnected,
    loginWithWallet,
    registerWithWallet,
  };
};
```

**src/hooks/useRole.ts**
```typescript
import { useMemo } from 'react';
import { useAuth } from './useAuth';
import { Permission, UserRole } from '../types/auth';
import { getRoleDisplayName, canAccessResource } from '../utils/roleUtils';

export const useRole = () => {
  const { user, hasPermission, hasAnyPermission } = useAuth();

  const roleInfo = useMemo(() => {
    if (!user) {
      return {
        role: null,
        displayName: 'Not authenticated',
        permissions: [],
      };
    }

    return {
      role: user.role,
      displayName: getRoleDisplayName(user.role),
      permissions: [], // Could be enhanced to return actual permissions
    };
  }, [user]);

  const canAccess = useMemo(() => ({
    invoices: {
      create: hasPermission(Permission.CREATE_INVOICE),
      viewOwn: hasPermission(Permission.VIEW_OWN_INVOICES),
      viewAll: hasPermission(Permission.VIEW_ALL_INVOICES),
      edit: hasPermission(Permission.EDIT_INVOICE),
      delete: hasPermission(Permission.DELETE_INVOICE),
    },
    users: {
      manage: hasPermission(Permission.MANAGE_USERS),
      viewProfiles: hasPermission(Permission.VIEW_USER_PROFILES),
    },
    system: {
      admin: hasPermission(Permission.SYSTEM_ADMIN),
      auditLogs: hasPermission(Permission.AUDIT_LOGS),
      export: hasPermission(Permission.EXPORT_DATA),
      import: hasPermission(Permission.IMPORT_DATA),
    },
  }), [hasPermission]);

  const checkResourceAccess = (
    resourceOwnerId: string,
    requiredPermission: Permission
  ): boolean => {
    if (!user) return false;
    
    return canAccessResource(
      user.role,
      resourceOwnerId,
      user.id,
      requiredPermission
    );
  };

  const isRole = (role: UserRole): boolean => {
    return user?.role === role;
  };

  const hasAnyRole = (roles: UserRole[]): boolean => {
    if (!user) return false;
    return roles.includes(user.role);
  };

  return {
    ...roleInfo,
    canAccess,
    checkResourceAccess,
    isRole,
    hasAnyRole,
    hasPermission,
    hasAnyPermission,
  };
};
```

**src/components/auth/RoleGuard.tsx**
```typescript
import React, { ReactNode } from 'react';
import { Permission, UserRole } from '../../types/auth';
import { useRole } from '../../hooks/useRole';

interface RoleGuardProps {
  children: ReactNode;
  permissions?: Permission[];
  roles?: UserRole[];
  requireAll?: boolean;
  fallback?: ReactNode;
  resourceOwnerId?: string;
  requiredPermission?: Permission;
}

export const RoleGuard: React.FC<RoleGuardProps> = ({
  children,
  permissions = [],
  roles = [],
  requireAll = false,
  fallback = <div className="access-denied">Access denied</div>,
  resourceOwnerId,
  requiredPermission,
}) => {
  const { hasPermission, hasAnyPermission, hasAnyRole, checkResourceAccess } = useRole();

  // Check resource-specific access
  if (resourceOwnerId && requiredPermission) {
    const canAccessResource = checkResourceAccess(resourceOwnerId, requiredPermission);
    if (!canAccessResource) {
      return <>{fallback}</>;
    }
  }

  // Check role-based access
  if (roles.length > 0) {
    const hasRequiredRole = hasAnyRole(roles);
    if (!hasRequiredRole) {
      return <>{fallback}</>;
    }
  }

  // Check permission-based access
  if (permissions.length > 0) {
    const hasRequiredPermissions = requireAll
      ? permissions.every(permission => hasPermission(permission))
      : hasAnyPermission(permissions);
    
    if (!hasRequiredPermissions) {
      return <>{fallback}</>;
    }
  }

  return <>{children}</>;
};

// Convenience components for common use cases
export const AdminOnly: React.FC<{ children: ReactNode; fallback?: ReactNode }> = ({
  children,
  fallback,
}) => (
  <RoleGuard roles={[UserRole.ADMIN]} fallback={fallback}>
    {children}
  </RoleGuard>
);

export const HealthcareProviderOnly: React.FC<{ children: ReactNode; fallback?: ReactNode }> = ({
  children,
  fallback,
}) => (
  <RoleGuard roles={[UserRole.HEALTHCARE_PROVIDER]} fallback={fallback}>
    {children}
  </RoleGuard>
);

export const PatientOnly: React.FC<{ children: ReactNode; fallback?: ReactNode }> = ({
  children,
  fallback,
}) => (
  <RoleGuard roles={[UserRole.PATIENT]} fallback={fallback}>
    {children}
  </RoleGuard>
);
```

**src/components/auth/LoginForm.tsx**
```typescript
import React, { useState } from 'react';
import { useAuth } from '../../hooks/useAuth';
import { UserRole } from '../../types/auth';

interface LoginFormProps {
  onSuccess?: () => void;
  onError?: (error: string) => void;
}

export const LoginForm: React.FC<LoginFormProps> = ({ onSuccess, onError }) => {
  const { loginWithWallet, registerWithWallet, isWalletConnected, isLoading, error } = useAuth();
  const [isRegistering, setIsRegistering] = useState(false);
  const [formData, setFormData] = useState({
    role: UserRole.PATIENT,
    email: '',
    name: '',
    organizationId: '',
  });

  const handleLogin = async () => {
    try {
      await loginWithWallet();
      onSuccess?.();
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Login failed';
      onError?.(errorMessage);
    }
  };

  const handleRegister = async () => {
    try {
      const additionalData: any = {};
      if (formData.email) additionalData.email = formData.email;
      if (formData.name) additionalData.name = formData.name;
      if (formData.organizationId) additionalData.organizationId = formData.organizationId;

      await registerWithWallet(formData.role, additionalData);
      onSuccess?.();
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Registration failed';
      onError?.(errorMessage);
    }
  };

  const handleInputChange = (e: React.ChangeEvent<HTMLInputElement | HTMLSelectElement>) => {
    const { name, value } = e.target;
    setFormData(prev => ({ ...prev, [name]: value }));
  };

  if (!isWalletConnected) {
    return (
      <div className="login-form">
        <div className="wallet-required">
          <h3>Wallet Connection Required</h3>
          <p>Please connect your wallet to access the Medical Invoice Storage system.</p>
        </div>
      </div>
    );
  }

  return (
    <div className="login-form">
      <div className="form-header">
        <h2>{isRegistering ? 'Register' : 'Login'}</h2>
        <p>
          {isRegistering 
            ? 'Create a new account with your wallet' 
            : 'Sign in with your wallet'
          }
        </p>
      </div>

      {error && (
        <div className="error-message">
          {error}
        </div>
      )}

      {isRegistering && (
        <div className="registration-fields">
          <div className="form-group">
            <label htmlFor="role">Role</label>
            <select
              id="role"
              name="role"
              value={formData.role}
              onChange={handleInputChange}
              required
            >
              <option value={UserRole.PATIENT}>Patient</option>
              <option value={UserRole.HEALTHCARE_PROVIDER}>Healthcare Provider</option>
              <option value={UserRole.INSURANCE_COMPANY}>Insurance Company</option>
              <option value={UserRole.AUDITOR}>Auditor</option>
            </select>
          </div>

          <div className="form-group">
            <label htmlFor="name">Full Name</label>
            <input
              type="text"
              id="name"
              name="name"
              value={formData.name}
              onChange={handleInputChange}
              placeholder="Enter your full name"
            />
          </div>

          <div className="form-group">
            <label htmlFor="email">Email</label>
            <input
              type="email"
              id="email"
              name="email"
              value={formData.email}
              onChange={handleInputChange}
              placeholder="Enter your email"
            />
          </div>

          {(formData.role === UserRole.HEALTHCARE_PROVIDER || 
            formData.role === UserRole.INSURANCE_COMPANY) && (
            <div className="form-group">
              <label htmlFor="organizationId">Organization ID</label>
              <input
                type="text"
                id="organizationId"
                name="organizationId"
                value={formData.organizationId}
                onChange={handleInputChange}
                placeholder="Enter organization identifier"
              />
            </div>
          )}
        </div>
      )}

      <div className="form-actions">
        <button
          type="button"
          onClick={isRegistering ? handleRegister : handleLogin}
          disabled={isLoading}
          className="primary-button"
        >
          {isLoading 
            ? 'Processing...' 
            : isRegistering 
              ? 'Register with Wallet' 
              : 'Login with Wallet'
          }
        </button>
      </div>

      <div className="form-footer">
        <button
          type="button"
          onClick={() => setIsRegistering(!isRegistering)}
          className="link-button"
        >
          {isRegistering 
            ? 'Already have an account? Login' 
            : "Don't have an account? Register"
          }
        </button>
      </div>
    </div>
  );
};
```

**src/components/auth/RoleSelector.tsx**
```typescript
import React, { useState } from 'react';
import { UserRole } from '../../types/auth';
import { useAuth } from '../../hooks/useAuth';
import { getRoleDisplayName } from '../../utils/roleUtils';

interface RoleSelectorProps {
  onRoleChange?: (role: UserRole) => void;
  disabled?: boolean;
}

export const RoleSelector: React.FC<RoleSelectorProps> = ({
  onRoleChange,
  disabled = false,
}) => {
  const { user, updateUserRole, isLoading } = useAuth();
  const [selectedRole, setSelectedRole] = useState<UserRole>(user?.role || UserRole.PATIENT);
  const [isUpdating, setIsUpdating] = useState(false);

  const handleRoleChange = async (newRole: UserRole) => {
    if (newRole === user?.role) return;

    setIsUpdating(true);
    try {
      await updateUserRole(newRole);
      setSelectedRole(newRole);
      onRoleChange?.(newRole);
    } catch (error) {
      console.error('Failed to update role:', error);
      // Reset to current role on error
      setSelectedRole(user?.role || UserRole.PATIENT);
    } finally {
      setIsUpdating(false);
    }
  };

  const availableRoles = [
    UserRole.PATIENT,
    UserRole.HEALTHCARE_PROVIDER,
    UserRole.INSURANCE_COMPANY,
    UserRole.AUDITOR,
  ];

  if (!user) {
    return null;
  }

  return (
    <div className="role-selector">
      <div className="current-role">
        <label>Current Role:</label>
        <span className={`role-badge role-${user.role}`}>
          {getRoleDisplayName(user.role)}
        </span>
      </div>

      <div className="role-selection">
        <label htmlFor="role-select">Change Role:</label>
        <select
          id="role-select"
          value={selectedRole}
          onChange={(e) => setSelectedRole(e.target.value as UserRole)}
          disabled={disabled || isLoading || isUpdating}
        >
          {availableRoles.map((role) => (
            <option key={role} value={role}>
              {getRoleDisplayName(role)}
            </option>
          ))}
        </select>

        <button
          type="button"
          onClick={() => handleRoleChange(selectedRole)}
          disabled={disabled || isLoading || isUpdating || selectedRole === user.role}
          className="update-role-button"
        >
          {isUpdating ? 'Updating...' : 'Update Role'}
        </button>
      </div>

      <div className="role-info">
        <small>
          Note: Changing your role may affect your access to certain features. 
          This action may require administrator approval.
        </small>
      </div>
    </div>
  );
};
```

### TEST FILES

**src/__tests__/utils/roleUtils.test.tsx**
```typescript
import { UserRole, Permission } from '../../types/auth';
import {
  getRolePermissions,
  hasPermission,
  hasAnyPermission,
  getRoleDisplayName,
  canAccessResource,
} from '../../utils/roleUtils';

describe('roleUtils', () => {
  describe('getRolePermissions', () => {
    it('should return correct permissions for PATIENT role', () => {
      const permissions = getRolePermissions(UserRole.PATIENT);
      expect(permissions).toContain(Permission.VIEW_OWN_INVOICES);
      expect(permissions).toContain(Permission.EXPORT_DATA);
      expect(permissions).not.toContain(Permission.CREATE_INVOICE);
    });

    it('should return correct permissions for ADMIN role', () => {
      const permissions =
```
